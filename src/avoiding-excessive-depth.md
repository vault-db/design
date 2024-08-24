# Avoiding excessively deep graphs

In VaultDB, operations will tend to have short dependency chains, typically with
only two levels, for example a `put()` depending on a set of `link()` calls.
`remove()` and `prune()` graphs may be deeper but these will normally be
executed rarely relative to `update()`. Despite this, it is possible to form
highly sub-optimal plans by following the rules we have so far. For example, we
could form a chain like the following:

    Shard │
          │   ┌────┐
        A │   │ w1 │
          │   └───\┘
          │        \
          │        ┌\──────────┐
        B │        │ w2     w3 │
          │        └──────────\┘
          │                    \
          │                    ┌\──────────┐
        C │                    │ w4     w5 │
          │                    └──────────\┘
          │                                \
          │                                ┌\──────────┐
        D │                                │ w6     w7 │
          │                                └──────────\┘
          │                                            \
          │                                            ┌\───┐
        E │                                            │ w8 │
          │                                            └────┘

This is constructed by following the rules for _w2_ depending on _w1_, _w4_ on
_w3_ and so on, and placing each new operation in the earliest existing group
for its shard. To complete eight operations we have _N_ = 5 and _D_ = 5, the
graph is completely sequential. While reducing _N_ is desirable, it is also good
to run requests in parallel where possible.

One way we could achieve this is by deciding to place _w5_ in a new group
_before_ that containing _w4_, giving us the following plan with _N_ = 6 and _D_
= 3.

    Shard │
          │   ┌────┐
        A │   │ w1 │
          │   └───\┘
          │        \
          │        ┌\──────────┐
        B │        │ w2     w3 │
          │        └──────────\┘
          │                    \
          │   ┌────┐           ┌\───┐
        C │   │ w5 │           │ w4 │
          │   └───\┘           └────┘
          │        \
          │        ┌\──────────┐
        D │        │ w6     w7 │
          │        └──────────\┘
          │                    \
          │                    ┌\───┐
        E │                    │ w8 │
          │                    └────┘

We still have some batched operations, but the graph is flatter and likely to
take less time to execute. It also allows for opportunistic merging; if the
request for group [_w5_] fails and must be retried, and we retry it after the
request for [_w2_, _w3_] completes, we can then fold _w5_ into the request for
[_w4_] because all the dependencies for _w4_ have completed. This is very much
a niche optimisation though, as normally shards have multiple groups as a result
of their dependencies _preventing_ the groups from merging, so a group is only
executed if all its previous groups were successfully committed. This dynamic
merging is only possible for disconnected subsets of the graph.

How would we implement this? We could keep track if the depth position of each
group, that is the length of the dependency chain to the left of a given group.
[_w1_] has depth 0 as it has no dependencies, [_w2_, _w3_] is depth 1, and
[_w4_] has depth 2. When we come to consider _w5_, we may decide not to add it
to a group of depth 2, but instead start a new depth-0 group for the same shard.

Whether to allow leaf operations to join depth-1 groups is a heuristic choice.
If we only place leaf operations in depth-0 groups, then we'd get the following
plan up to _w4_:

    Shard │
          │       ┌────┐                    N = 4, D = 2
        A │       │ w1 │
          │       └───\┘
          │            \
          │   ┌────┐   ┌\───┐
        B │   │ w3 │   │ w2 │
          │   └───\┘   └────┘
          │        \
          │        ┌\───┐
        C │        │ w4 │
          │        └────┘

Whereas if we allow leaf operations in depth-1 groups, then we get some group
merging without the long chain of stacked requests.

    Shard │
          │   ┌────┐                        N = 3, D = 3
        A │   │ w1 │
          │   └───\┘
          │        \
          │        ┌\──────────┐
        B │        │ w2     w3 │
          │        └──────────\┘
          │                    \
          │                    ┌\───┐
        C │                    │ w4 │
          │                    └────┘

This is especially useful in the case of "crossed" dependencies where we want to
update two documents where each document's parent directory is in the same shard
as the other document. For example:

- Shard _A_ contains document item `/alice/note` and directory item `/bob/`
- Shard _B_ contains document item `/bob/note` and directory item `/alice/`

We cannot complete these updates via a single write to each shard without
violating causality, but we can accomplish it in three requests:

- _w1_ represents `link('/alice/', 'note')` and targets shard `B`
- _w2_ represents `put('/alice/note')` and targets shard `A`; it depends on _w1_
- _w3_ represents `link('/bob/', 'note')` and targets shard `A`
- _w4_ represents `put('/bob/note')` and targets shard `B`; it depends on _w3_

These operations produce this graph if we allow leaf operations in depth-1
groups:

    Shard │
          │        ┌───────────┐
        A │        │ w2     w3 │
          │        └/─────────\┘
          │        /           \
          │   ┌───/┐           ┌\───┐
        B │   │ w1 │           │ w4 │
          │   └────┘           └────┘

If we don't allow leaf operations in depth-1 groups, then we still end up with
an _N_ = 3 graph because even though we will place _w3_ in its own group before
_w2_, we will merge _w4_ into the group with _w1_.

    Shard │
          │   ┌────┐           ┌────┐
        A │   │ w3 │           │ w2 │
          │   └───\┘           └/───┘
          │        \           /
          │        ┌\─────────/┐
        B │        │ w4     w1 │
          │        └───────────┘

The leaf operation _w1_ has ended up in a depth-1 group because we merged _w4_
with it, and _w4_ depends on _w3_. So we may as well allow leaf operations to be
placed in depth-1 groups to begin with.

This works because _w1_ and _w4_ target the same shard; if they targeted
different shards then we'd get the stair-step pattern we saw above. The
particular way the operations are grouped and ordered is affected by whether we
allow leaf operations into depth-1 groups or not, and this will affect how
future operations merge into the graph. Again, we don't want to try every
possible combination but instead pick rules that produce a reasonable plan with
a single pass over the set of operations.

We may also want to place restrictions on merging non-leaf operations into
existing groups. For example, consider a variation of an earlier example where
we've built the following graph, and we're about to add _w8_ which targets _C_
and depends on _w7_.

    Shard │
          │   ┌────┐
        A │   │ w5 │
          │   └───\┘
          │        \
          │        ┌\──────────┐
        B │        │ w6     w7 │
          │        └──────────\┘
          │                    \
          │   ┌────┐           ┌\   ┐
        C │   │ w1 │             w8
          │   └───\┘           └    ┘
          │        \
          │        ┌\──────────┐
        D │        │ w2     w3 │
          │        └──────────\┘
          │                    \
          │                    ┌\───┐
        E │                    │ w4 │
          │                    └────┘

If we merge _w8_ into the [_w1_] group, that will have the effect of delaying
all the groups for shards _C_, _D_ and _E_, and increase the graph's total depth
from 3 to 5. We may decide to place _w8_ in its own group of depth 2 than to
merge it into a depth-0 group, thereby increasing the depth of a bunch of
existing groups.

What if the operations on shards _D_ and _E_ didn't exist?

    Shard │
          │   ┌────┐
        A │   │ w2 │
          │   └───\┘
          │        \
          │        ┌\──────────┐
        B │        │ w3     w4 │
          │        └──────────\┘
          │                    \
          │   ┌────┐           ┌\   ┐
        C │   │ w1 │             w5
          │   └────┘           └    ┘

In this case, we'd probably want to merge _w5_ into the [_w1_] group, as doing
that doesn't increase the graph's total depth and it reduces the number of
groups by one. It's hard to make this call while building the graph, because we
don't know what further operations might be built on top of _w1_. Perhaps this
is an operation we could apply when we've finished building the graph, to merge
adjacent groups as long as one does not depend on the other, and merging them
doesn't increase the graph's total depth.

For example, if _w5_ is the final operation and we end up with a graph like
this:

    Shard │
          │   ┌────┐
        A │   │ w2 │
          │   └───\┘
          │        \
          │        ┌\──────────┐
        B │        │ w3     w4 │
          │        └──────────\┘
          │                    \
          │   ┌────┐           ┌\───┐
        C │   │ w1 │           │ w5 │
          │   └────┘           └────┘

Then we can reduce _N_ without affecting _D_ by merging the [_w1_] and [_w5_]
groups:

    Shard │
          │   ┌────┐
        A │   │ w2 │
          │   └───\┘
          │        \
          │        ┌\──────────┐
        B │        │ w3     w4 │
          │        └──────────\┘
          │                    \
          │                    ┌\──────────┐
        C │                    │ w5     w1 │
          │                    └───────────┘

But suppose further operations are added after _w5_ that depend on _w1_ and
target another shard:

    Shard │
          │   ┌────┐
        A │   │ w2 │
          │   └───\┘
          │        \
          │        ┌\──────────┐
        B │        │ w3     w4 │
          │        └──────────\┘
          │                    \
          │   ┌────┐           ┌\───┐
        C │   │ w1 │           │ w5 │
          │   └───\┘           └────┘
          │        \
          │        ┌\───┐
        D │        │ w6 │
          │        └────┘

In this case, merging the [_w1_] and [_w5_] groups would reduce _N_ by 1, but
would also increase _D_ by 1:

    Shard │
          │   ┌────┐
        A │   │ w2 │
          │   └───\┘
          │        \
          │        ┌\──────────┐
        B │        │ w3     w4 │
          │        └──────────\┘
          │                    \
          │                    ┌\──────────┐
        C │                    │ w5     w1 │
          │                    └──────────\┘
          │                                \
          │                                ┌\───┐
        D │                                │ w6 │
          │                                └────┘

This suggests that it would be profitable to avoid merging groups whose depth
differs by 2 or more (for example adding _w5_ to the [_w1_] group) while adding
operations to the graph. Once all operations have been added, we could look for
opportunities to merge groups for the same shard, as long as this does not
increase the graph's total depth, or increases it by less than some threshold if
we believe that reducing _N_ is worth increasing _D_ somewhat.

We could also consider approaches based on building the graph only after we have
all the dependency information for all writes, rather than attempting to build
the graph incrementally. Any graph-building algorithm will necessarily involve
an incremental phase to slot operations into the graph one by one, but can we
apply any analysis to the operations before starting this process that leads to
a better result?

For example, we could decide to sort the operations by their depth: leaf
operations are depth 0, operations that depend only on leaves are depth 1, and
so on. Take our stair-step plan from above:

    Shard │
          │   ┌────┐
        A │   │ w1 │
          │   └───\┘
          │        \
          │        ┌\──────────┐
        B │        │ w2     w3 │
          │        └──────────\┘
          │                    \
          │                    ┌\──────────┐
        C │                    │ w4     w5 │
          │                    └──────────\┘
          │                                \
          │                                ┌\──────────┐
        D │                                │ w6     w7 │
          │                                └──────────\┘
          │                                            \
          │                                            ┌\───┐
        E │                                            │ w8 │
          │                                            └────┘

Imagine we instead try to build a graph by adding all the depth-0 operations to
it first. These have no dependencies and will all produce a single group for
their respective shard.

    Shard │
          │   ┌────┐
        A │   │ w1 │
          │   └────┘
          │
          │   ┌────┐
        B │   │ w3 │
          │   └────┘
          │
          │   ┌────┐
        C │   │ w5 │
          │   └────┘
          │
          │   ┌────┐
        D │   │ w7 │
          │   └────┘
          │
          │
        E │
          │

Then we could handle the depth-1 operations. _w2_ depends on _w1_ and targets
shard _B_, so we can merge it into the [_w3_] group, increasing that group's
depth from 0 to 1. _w4_ depends on _w3_ and targets _C_, forcing the creation of
a depth-2 group. We don't merge _w4_ into the [_w5_] group because that would
increase the group's depth by 2.

    Shard │
          │   ┌────┐
        A │   │ w1 │
          │   └───\┘
          │        \
          │        ┌\──────────┐
        B │        │ w2     w3 │
          │        └──────────\┘
          │                    \
          │   ┌────┐           ┌\───┐
        C │   │ w5 │           │ w4 │
          │   └────┘           └────┘
          │
          │   ┌────┐
        D │   │ w7 │
          │   └────┘
          │
          │
        E │
          │

We can see how following this pattern produces a similar result to what we saw
above. It doesn't seem like sorting by operation depth produced a radically
different approach; graph-building decisions are still guided by the relative
depth of write groups. Since dependency chains in VaultDB are typically short
and msot operations have a depth of either 0 or 1, sorting them by depth doesn't
meaningfully change the graph building process. It just adds delay to the start
of the process, since we can't start building the graph until we have
information for all operations and we've sorted them. It also prevents graphs
being dynamically updated or optimised during execution, which is necessarily
for dealing with failed requests.

So far the best candidates for constraints to add to our rule set are:

- Do not add a leaf operation to the leftmost existing group if that group's
  current depth is greater than 1. Instead, create a new depth-0 group for the
  target shard.

- Do not add any operation to an existing group if doing so would increase the
  group's depth by more than 1. Instead, start a new group for the operation.

- Once all operations are known, attempt to merge any groups for a shard that
  are not mutually dependent. Merge groups as long as it does not increase the
  total graph depth unacceptably.
