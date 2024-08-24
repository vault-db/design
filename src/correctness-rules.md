# Correctness rules

Let's start with an example that models the above `update()` call:

- Operation _w1_ targets shard _B_ and has no dependencies
- Operation _w2_ targets shard _A_ and has no dependencies
- Operation _w3_ targets shard _A_ and depends on _w1_ and _w2_.

We can imagine building up a graph of these operations while trying to group
operations on the same shard. We'll start by placing _w1_ in the graph in a new
group for shard _B_.

    Shard │
          │
        A │
          │
          │
          │   ┌────┐
        B │   │ w1 │
          │   └────┘

Next, we place _w2_, which targets shard _A_, so we create a new group for that
shard:

    Shard │
          │   ┌────┐
        A │   │ w2 │
          │   └────┘
          │
          │   ┌────┐
        B │   │ w1 │
          │   └────┘

Finally, we need to add _w3_, which depends on the other two operations. It must
therefore be executed either _after_ or _at the same time as_ these operations
to achieve safety. It targets shard _A_, so our question is: can we put it in
the existing group containing _w1_, or must we create a new group? In this case,
_w3_ depends directly on _w1_ and it is safe to place it in the same group,
meaning _w1_ and _w3_ will be applied in the same write request.

We add _w3_ to the graph in the group with _w1_ and make dependency links
between the operations:

    Shard │
          │   ┌────────────┐
        A │   │ w2 ---- w3 │
          │   └────────/───┘
          │           /
          │      ┌───/┐
        B │      │ w1 │
          │      └────┘

The finished graph has a dependency between the [_w2_, _w3_] group and the
[_w1_] group, forcing the writes to be executed in that order:

    Shard │
          │                         ┌─────────────────┐
        A │                         │ WRITE ░░░░░░░░░░│
          │                         ├─────────────────┘
          │                         └─ w2, w3
          │   ┌─────────────────┐
        B │   │ WRITE ░░░░░░░░░░│
          │   ├─────────────────┘
              └─ w1

Now imagine that we add a fourth operation _w4_ that targets _B_ and depends on
_w3_. Can we put it in the same group as _w1_? No, we cannot: this would mean
executing _w1_ and _w4_ in one write, and then _w2_ and _w3_ in another. _w3_
happens after _w4_, or possibly not at all if the second write fails, so we've
violated the dependency order of _w4_ on _w3_. Therefore _w4_ needs to be put in
its own group, separate from _w1_.

    Shard │
          │   ┌────────────┐
        A │   │ w2 ---- w3 │
          │   └────────/──\┘
          │           /    \
          │      ┌───/┐    ┌\───┐
        B │      │ w1 │    │ w4 │
          │      └────┘    └────┘

That gives us the following write sequence:

    Shard │
          │                       ┌─────────────────┐
        A │                       │ WRITE ░░░░░░░░░░│
          │                       ├─────────────────┘
          │                       └─ w2, w3
          │   ┌─────────────────┐                     ┌─────────────────┐
        B │   │ WRITE ░░░░░░░░░░│                     │ WRITE ░░░░░░░░░░│
          │   ├─────────────────┘                     ├─────────────────┘
              └─ w1                                   └─ w4

This small example has given us some useful constraints for building graphs:

- If no write groups exist for a shard, a new group must be created.

- If operations _P_ and _Q_ target the same shard, and _P_ depends directly on
  _Q_, then _P_ may be placed in the same group as _Q_, but must not be placed
  in any earlier group.

- If an operation _P_ depends _indirectly_ on another operation _Q_, that is it
  depends on _Q_ via an operation on another shard, then _P_ must be placed in a
  later group than _Q_.

- Otherwise, operations may be assigned to any group we choose without breaking
  causality, assuming we record dependencies to force the groups to execute in
  the correct order.
