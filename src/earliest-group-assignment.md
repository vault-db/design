# Assignment to the earliest group

If there are multiple groups that we're allowed to add an operation to, which
one do we choose? In general, it seems wise to choose the earliest-scheduled
group, that is the leftmost one in the graph. To see why, consider the following
example. Imagine we've built up the following graph, where:

- The operation _w1_ targets shard _B_.
- _w2_ targets shard _A_ and depends on _w1_.
- _w3_ targets _B_ and has no dependencies.
- _w4_ targets _C_ and depends on _w3_.
- _w5_ targets _B_ and depends on _w4_.

This gives us the following graph according to the above rules:

    Shard │
          │                 ┌────┐
        A │          .------- w2 │
          │         /       └────┘
          │        /
          │   ┌───/────────┐    ┌────┐
        B │   │ w1      w3 │    │ w5 │
          │   └───────────\┘    └/───┘
          │                \    /
          │                ┌\──/┐
        C │                │ w4 │
          │                └────┘

_w1_ and _w3_ have been merged into a group because they do not depend on each
other, so it is safe to execute them together as long as the other writes start
_after_ the write to _B_ is complete. The indirect dependency of _w5_ on _w3_
means it cannot be merged with _w3_ and must start a new group, otherwise it
would be committed before _w4_.

Now say we have another operation targeting _B_, _w6_. It has no dependencies
and we decide to merge it into the group with _w5_.

    Shard │
          │                 ┌────┐
        A │          .------- w2 │
          │         /       └────┘
          │        /
          │   ┌───/────────┐    ┌────────────┐
        B │   │ w1      w3 │    │ w5      w6 │
          │   └───────────\┘    └/───────────┘
          │                \    /
          │                ┌\──/┐
        C │                │ w4 │
          │                └────┘

Next we add _w7_, targeting _A_ and depending on _w6_. We can merge this with
_w2_ safely, with the effect that this group is now scheduled after all the
others.

    Shard │
          │                          ┌────────────┐
        A │          .---------------- w2      w7 │
          │         /                └────────/───┘
          │        /                         /
          │   ┌───/────────┐    ┌───────────/┐
        B │   │ w1      w3 │    │ w5      w6 │
          │   └───────────\┘    └/───────────┘
          │                \    /
          │                ┌\──/┐
        C │                │ w4 │
          │                └────┘

Finally we add _w8_ which targets _B_ and depends on _w4_ and _w7_. Its indirect
dependencies on _w3_ and _w6_ mean it cannot be merged into any existing group
without violating causality, and must be put in a group by itself.

    Shard │
          │                          ┌────────────┐
        A │          .---------------- w2      w7 │
          │         /                └────────/──\┘
          │        /                         /    \
          │   ┌───/────────┐    ┌───────────/┐    ┌\───┐
        B │   │ w1      w3 │    │ w5      w6 │    │ w8 │
          │   └───────────\┘    └/───────────┘    └/───┘
          │                \    /                 /
          │                ┌\──/┐                /
        C │                │ w4 ----------------'
          │                └────┘

This gives us the following execution for this set of writes, which preserves
the required ordering so that we do not start writing an operation until after
all its dependencies are successfully committed.

    Shard │
          │                                                   ┌─────────────┐
        A │                                                   │ WRITE ░░░░░░│
          │                                                   ├─────────────┘
          │                                                   └─ w2, w7
          │   ┌─────────────┐                 ┌─────────────┐                 ┌─────────────┐
        B │   │ WRITE ░░░░░░│                 │ WRITE ░░░░░░│                 │ WRITE ░░░░░░│
          │   ├─────────────┘                 ├─────────────┘                 ├─────────────┘
          │   └─ w1, w3                       └─ w5, w6                       └─ w8
          │                   ┌─────────────┐
        C │                   │ WRITE ░░░░░░│
          │                   ├─────────────┘
                              └─ w4

However, this execution is not optimal; we perform a total of five writes, all
sequentially, and _w2_ is scheduled much later than necessary, considering it
only depends on _w3_ being committed. What if instead we put _w6_ in the first
group, with _w1_ and _w3_?

    Shard │
          │                 ┌────┐
        A │          .------- w2 │
          │         /       └────┘
          │        /
          │   ┌───/────────────────┐    ┌────┐
        B │   │ w1      w3      w6 │    │ w5 │
          │   └───────────\────────┘    └/───┘
          │                \            /
          │                ┌\───┐      /
        C │                │ w4 ------'
          │                └────┘

Now, _w7_ can again be merged with _w2_, but now we do not delay _w2_ until
after _w5_ is committed.

    Shard │
          │                 ┌───────────┐
        A │          .------- w2     w7 │
          │         /       └───────/───┘
          │        /               /
          │   ┌───/───────────────/┐    ┌────┐
        B │   │ w1      w3      w6 │    │ w5 │
          │   └───────────\────────┘    └/───┘
          │                \            /
          │                ┌\───┐      /
        C │                │ w4 ------'
          │                └────┘

When we plan _w8_, we can now put it in the group with _w5_, on which it does
not depend. We cannot place _w8_ any earlier because of its indirect
dependencies on _w3_ and _w6_.

    Shard │
          │                 ┌───────────┐
        A │          .------- w2     w7 ------.
          │         /       └───────/───┘      \
          │        /               /            \
          │   ┌───/───────────────/┐    ┌────────\───┐
        B │   │ w1      w3      w6 │    │ w5      w8 │
          │   └───────────\────────┘    └/───────/───┘
          │                \            /       /
          │                ┌\───┐      /       /
        C │                │ w4 ------'-------'
          │                └────┘

We now have four writes rather than five, and two of them can be performed
concurrently whereas our previous plan required all writes to happen
sequentially, so the depth has been reduced from five to three.

    Shard │
          │                     ┌───────────────┐
        A │                     │ WRITE ░░░░░░░░│
          │                     ├───────────────┘
          │                     └─ w2, w7
          │   ┌───────────────┐                   ┌───────────────┐
        B │   │ WRITE ░░░░░░░░│                   │ WRITE ░░░░░░░░│
          │   ├───────────────┘                   ├───────────────┘
          │   └─ w1, w3, w6                       └─ w5, w8
          │                     ┌───────────────┐
        C │                     │ WRITE ░░░░░░░░│
          │                     ├───────────────┘
                                └─ w4

This suggests that when trying to merge an operation into an existing group, the
earliest or "leftmost" eligible group in the graph should be chosen, to reduce
the likely total depth of the graph. For "leaf" operations with no dependencies,
this means picking the first group at all times, but non-leaf operations must be
placed in groups to the right of any operations they depend on. In general,
operations should be scheduled as early as possible, but no earlier.

There is another important property at work here. Look again at the graph state
before we add _w6_:

    Shard │
          │                 ┌────┐
        A │          .------- w2 │
          │         /       └────┘
          │        /
          │   ┌───/────────┐    ┌────┐
        B │   │ w1      w3 │    │ w5 │
          │   └───────────\┘    └/───┘
          │                \    /
          │                ┌\──/┐
        C │                │ w4 │
          │                └────┘

The [_w1_, _w3_] group is already depended upon by the [_w2_] group, and so
adding _w6_ and _w7_ to these groups respectively does not create any new
inter-group dependencies that would block admission of new operations into the
first _B_ group. It was already impossible for anything depending on the [_w2_]
group to enter the [_w1_, _w3_] group, and the addition of _w6_ and _w7_ has not
changed that.

    Shard │
          │                 ┌───────────┐
        A │          .------- w2     w7 │
          │         /       └───────/───┘
          │        /               /
          │   ┌───/───────────────/┐    ┌────┐
        B │   │ w1      w3      w6 │    │ w5 │
          │   └───────────\────────┘    └/───┘
          │                \            /
          │                ┌\───┐      /
        C │                │ w4 ------'
          │                └────┘

So it is tempting to try to plan execution by avoiding the creation of new
inter-group dependencies, if possible. However, if we're considering each
operation one at a time while building the graph, when _w6_ is added we do not
yet know what other operations might come to depend on it.

    Shard │
          │                 ┌────┐
        A │          .------- w2 │
          │         /       └────┘
          │        /
          │   ┌───/────────────────┐    ┌────┐
        B │   │ w1      w3      w6 │    │ w5 │
          │   └───────────\────────┘    └/───┘
          │                \            /
          │                ┌\───┐      /
        C │                │ w4 ------'
          │                └────┘

Doing this sort of analysis requires comparing whole dependency sets and their
possible intersections, and every possible combination of groups they could
form, rather than building the graph one operation at a time, and is likely to
be more computationally expensive and harder to define an algorithm for it. So
for now we will take the simpler option of preferring to place each operation in
the earliest possible group.
