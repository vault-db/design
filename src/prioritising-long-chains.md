# Prioritising the longest chains

Let's consider another example where we'd want to constrain merging based on
group depth. Imagine we have the following graph:

    Shard │
          │        ┌───────────┐
        A │        │ w2     w3 │
          │        └/─────────\┘
          │        /           \
          │   ┌───/┐           ┌\───┐
        B │   │ w1 │           │ w4 │
          │   └────┘           └───\┘
          │                         \
          │                         ┌\───┐
        C │                         │ w5 │
          │                         └────┘

Now say we have operation _w6_ targeting _C_, and it has no dependencies. We
decide not to merge it with the [_w5_] group, since that has depth 3, and we
instead start a new depth-0 group for it.

    Shard │
          │        ┌───────────┐
        A │        │ w2     w3 │
          │        └/─────────\┘
          │        /           \
          │   ┌───/┐           ┌\───┐
        B │   │ w1 │           │ w4 │
          │   └────┘           └───\┘
          │                         \
          │   ┌────┐                ┌\───┐
        C │   │ w6 │                │ w5 │
          │   └────┘                └────┘

Again, if no further operations are added to this graph, we'd consider merging
the [_w6_] and [_w5_] groups to reduce the number of groups without increasing
depth. But imagine instead that we now add _w7_, which targets _B_ and depends
on _w6_. Which group do we add _w7_ to?

_w7_ does not depend in any way on _w1_ or _w4_, so we can add it to either of
their groups. If we pick the earlier group, the [_w1_] one, then link that group
to [_w6_], we end up increasing the depth of the graph to 5.

    Shard │
          │                     ┌───────────┐
        A │                     │ w2     w3 │
          │                     └/─────────\┘
          │                     /           \
          │        ┌───────────/┐           ┌\───┐
        B │        │ w7      w1 │           │ w4 │
          │        └/───────────┘           └───\┘
          │        /                             \
          │   ┌───/┐                             ┌\───┐
        C │   │ w6 │                             │ w5 │
          │   └────┘                             └────┘

However, if we placed _w7_ in the [_w4_] group, then linking this group to
[_w6_] does not increase the graph's depth and it remains at 4.

    Shard │
          │        ┌───────────┐
        A │        │ w2     w3 │
          │        └/─────────\┘
          │        /           \
          │   ┌───/┐   ┌────────\───┐
        B │   │ w1 │   │ w7      w4 │
          │   └────┘   └/──────────\┘
          │            /            \
          │       ┌───/┐            ┌\───┐
        C │       │ w6 │            │ w5 │
          │       └────┘            └────┘

Therefore when merging a non-leaf operation, we should pick a group whose depth
is greater than that of any of the groups the operation depends on. Remember
that it's the depth of _groups_ that matters here for the shape of the graph,
not the depth of operations within their own local dependency relationships.
This graph produces the following partly parallelised write sequence, and has
_N_ = 5 and _D_ = 4.

    Shard │
          │                   ┌─────────────┐
        A │                   │ WRITE ░░░░░░│
          │                   ├─────────────┘
          │                   └─ w2, w3
          │   ┌─────────────┐                 ┌─────────────┐
        B │   │ WRITE ░░░░░░│                 │ WRITE ░░░░░░│
          │   ├─────────────┘                 ├─────────────┘
          │   └─ w1                           └─ w7, w4
          │   ┌─────────────┐                                 ┌─────────────┐
        C │   │ WRITE ░░░░░░│                                 │ WRITE ░░░░░░│
          │   ├─────────────┘                                 ├─────────────┘
              └─ w6                                           └─ w5

This is a shorter graph depth than if we'd decided to merge _w6_ with _w5_, thus
pushing _w7_ into its own group, or than if we'd merged _w7_ with _w1_ as
described above. Both these other graphs would be fully sequential.

The results we end up with are sensitive to the order in which operations are
added, and the merging decisions we make as a result. Imagine that in this
example we'd instead ended up merging _w1_ and _w4_, rather than _w2_ and _w3_,
which is perfectly allowed by their dependency relationships.

    Shard │
          │   ┌────┐           ┌────┐
        A │   │ w3 │           │ w2 │
          │   └───\┘           └/───┘
          │        \           /
          │        ┌\─────────/┐
        B │        │ w4     w1 │
          │        └───────────┘
          │
          │
        C │
          │

With the information up to _w4_, there's no reason to prefer this over the
previous plan, they're structurally equivalent. The difference only becomes
apparent when we add more operations.

We then add _w5_, creating a group of depth 2 that depends on the depth-1 [_w4_,
_w1_] group, and we add _w6_ in its own depth-0 group rather than adding it to
the depth-2 [_w5_] group.

    Shard │
          │   ┌────┐           ┌────┐
        A │   │ w3 │           │ w2 │
          │   └───\┘           └/───┘
          │        \           /
          │        ┌\─────────/┐
        B │        │ w4     w1 │
          │        └───\───────┘
          │             \
          │   ┌────┐    ┌\───┐
        C │   │ w6 │    │ w5 │
          │   └────┘    └────┘

Then we add _w7_, which we can merge into the [_w4_, _w1_] group, linking that
group to [_w6_] creating a graph with _N_ = 5 and _D_ = 3.

    Shard │
          │          ┌────┐          ┌────┐
        A │          │ w3 │          │ w2 │
          │          └───\┘          └/───┘
          │               \          /
          │        ┌───────\────────/┐
        B │        │ w7     w4    w1 │
          │        └/─────────\──────┘
          │        /           \
          │   ┌───/┐           ┌\───┐
        C │   │ w6 │           │ w5 │
          │   └────┘           └────┘

This graph has _N_ = 5 like the ones before it, but now performs more of the
writes in parallel. The difference isn't big enough to be worth trying every
possible grouping of operations, it's more important to have an algorithm that
can make some reasonable optimisations that runs in linear time relative to the
number of operations. Sometimes we'll miss a slightly more optimal solution, but
that's preferable to making the optimiser much more complicated or have
super-linear performance scaling.

    Shard │
          │   ┌───────────────┐                       ┌───────────────┐
        A │   │ WRITE ░░░░░░░░│                       │ WRITE ░░░░░░░░│
          │   ├───────────────┘                       ├───────────────┘
          │   └─ w3                                   └─ w2
          │                       ┌───────────────┐
        B │                       │ WRITE ░░░░░░░░│
          │                       ├───────────────┘
          │                       └─ w1, w4, w7
          │   ┌───────────────┐                       ┌───────────────┐
        C │   │ WRITE ░░░░░░░░│                       │ WRITE ░░░░░░░░│
          │   ├───────────────┘                       ├───────────────┘
              └─ w6                                   └─ w5

Is there a method we could use to convert the _D_ = 4 solution for this set of
operations into the _D_ = 3 graph? Consider the graph shape we ended up with for
_D_ = 4:

    Shard │
          │        ┌───────────┐
        A │        │ w2     w3 │
          │        └/─────────\┘
          │        /           \
          │   ┌───/┐   ┌────────\───┐
        B │   │ w1 │   │ w7      w4 │
          │   └────┘   └/──────────\┘
          │            /            \
          │       ┌───/┐            ┌\───┐
        C │       │ w6 │            │ w5 │
          │       └────┘            └────┘

We can observe that the longest dependency chain between single operations is
the one connecting _w3_, _w4_ and _w5_. Imagine that we construct a new graph
containing just those operations, and split out the other operations into their
own sub-graphs:

    Shard │                                     │
          │   ┌────┐                            │        ┌────┐
        A │   │ w3 │                            │        │ w2 │
          │   └───\┘                            │        └/───┘
          │        \                            │        /
          │        ┌\───┐                       │   ┌───/┐      ┌────┐
        B │        │ w4 │                       │   │ w1 │      │ w7 │
          │        └───\┘                       │   └────┘      └/───┘
          │             \                       │               /
          │             ┌\───┐                  │          ┌───/┐
        C │             │ w5 │                  │          │ w6 │
          │             └────┘                  │          └────┘

We can then add the remaining operations back into the graph on the left. We
start by merging _w1_ into the [_w4_] group, since _w1_ has no dependencies and
the [_w4_] group has depth 1. Then we create a new group for _w2_ which depends
on _w1_:

    Shard │                                     │
          │   ┌────┐           ┌────┐           │
        A │   │ w3 │           │ w2 │           │
          │   └───\┘           └/───┘           │
          │        \           /                │
          │        ┌\─────────/┐                │               ┌────┐
        B │        │ w4     w1 │                │               │ w7 │
          │        └───\───────┘                │               └/───┘
          │             \                       │               /
          │             ┌\───┐                  │          ┌───/┐
        C │             │ w5 │                  │          │ w6 │
          │             └────┘                  │          └────┘

This leaves _w6_ and _w7_ to merge into the main graph. _w6_ gets a new group,
rather than adding it to the [_w5_] group which has depth 2. _w7_ can be added
to the [_w4_, _w1_] group and linked to the [_w6_] group.

    Shard │
          │          ┌────┐          ┌────┐
        A │          │ w3 │          │ w2 │
          │          └───\┘          └/───┘
          │               \          /
          │        ┌───────\────────/┐
        B │        │ w7     w4    w1 │
          │        └/─────────\──────┘
          │        /           \
          │   ┌───/┐           ┌\───┐
        C │   │ w6 │           │ w5 │
          │   └────┘           └────┘

By extracting the longest dependency chain between individual operations and
building the graph by starting with that chain, we reduce the total depth of the
graph. Here's another example of a graph that could be optimised in this way:

    Shard │
          │   ┌────┐            ┌────┐
        A │   │ w1 │            │ w6 │
          │   └───\┘            └/──\┘
          │        \            /    \
          │        ┌\──────────/┐    ┌\───┐
        B │        │ w2      w5 │    │ w7 │
          │        └────────/───┘    └────┘
          │                /
          │           ┌───/┐
        C │           │ w4 │
          │           └/───┘
          │           /
          │      ┌───/┐
        D │      │ w3 │
          │      └────┘

The dominant chain in this graph is the set of operations [_w3_, _w4_, _w5_,
_w6_, _w7_]. If we rebuild the graph starting with these operations, and then
add _w1_ and _w2_ back in, we end up with the following graph where _w1_ and
_w2_ are in their own groups because the existing groups for their shards have
too high a depth.

    Shard │
          │   ┌────┐               ┌────┐
        A │   │ w1 │               │ w6 │
          │   └───\┘               └/──\┘
          │        \               /    \
          │        ┌\───┐     ┌───/┐    ┌\───┐
        B │        │ w2 │     │ w5 │    │ w7 │
          │        └────┘     └/───┘    └────┘
          │                   /
          │              ┌───/┐
        C │              │ w4 │
          │              └/───┘
          │              /
          │         ┌───/┐
        D │         │ w3 │
          │         └────┘

However we could then merge the [_w1_] and [_w6_] groups without increasing the
total depth of the graph; the [_w2_] and [_w7_] groups both have depth 4 after
this merge.

    Shard │
          │                  ┌──────────────┐
        A │                  │ w6        w1 │
          │                  └/──\─────────\┘
          │                  /    \         \
          │             ┌───/┐    ┌\───┐    ┌\───┐
        B │             │ w5 │    │ w7 │    │ w2 │
          │             └/───┘    └────┘    └────┘
          │             /
          │        ┌───/┐
        C │        │ w4 │
          │        └/───┘
          │        /
          │   ┌───/┐
        D │   │ w3 │
          │   └────┘

Then we can also merge the [_w7_] and [_w2_] groups, because these operations
are independent. We're left with this graph that's reduced _N_ by 1 compared to
the graph we started with.

    Shard │
          │                  ┌───────────┐
        A │                  │ w6     w1 │
          │                  └/──\──────\┘
          │                  /    \      \
          │             ┌───/┐    ┌\──────\───┐
        B │             │ w5 │    │ w7     w2 │
          │             └/───┘    └───────────┘
          │             /
          │        ┌───/┐
        C │        │ w4 │
          │        └/───┘
          │        /
          │   ┌───/┐
        D │   │ w3 │
          │   └────┘

This type of optimisation assumes the total graph depth will be dominated by one
or a few long operation chains; in VaultDB this won't be especially useful since
most writes involve chains of just two operations, but this could be useful for
applying this planning algorithm to other problems. It might be useful to make
this optimisation an optional task to perform once the initial graph is built,
and before attempting to merge groups.
