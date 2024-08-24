# Merging the final groups

For example, consider these two graphs, which are almost identical except that
in the first, _w6_ targets shard _D_ and in the second in targets _B_. In the
second graph, _w6_ has been placed in its own group rather than merging it into
the [_w1_] group, because according to the above discussion we should place _w6_
in a group whose depth is at least 1.

    Shard │
          │        ┌───────────┐
        A │        │ w2     w3 │
          │        └/─────────\┘
          │        /           \
          │   ┌───/┐            \
        B │   │ w1 │             \
          │   └────┘              \
          │                        \
          │   ┌────┐               ┌\───┐
        C │   │ w5 │               │ w4 │
          │   └───\┘               └────┘
          │        \
          │        ┌\───┐
        D │        │ w6 │
          │        └────┘

    Shard │
          │        ┌───────────┐
        A │        │ w2     w3 │
          │        └/─────────\┘
          │        /           \
          │   ┌───/┐    ┌────┐  \
        B │   │ w1 │    │ w6 │   \
          │   └────┘    └/───┘    \
          │             /          \
          │   ┌────┐   /           ┌\───┐
        C │   │ w5 ───'            │ w4 │
          │   └────┘               └────┘

In both these graphs, the [_w5_] and [_w4_] groups can be merged at the cost of
increasing _D_ from three to four. In the second graph it is possible to instead
merge the [_w1_] and [_w6_] groups, which also increases _D_ to four. That is,
we could end up with either of these graphs after performing group merging on
the second graph:

    Shard │
          │        ┌───────────┐
        A │        │ w2     w3 │
          │        └/─────────\┘
          │        /           \
          │   ┌───/┐            \              ┌────┐
        B │   │ w1 │             \             │ w6 │
          │   └────┘              \            └/───┘
          │                        \           /
          │                        ┌\─────────/┐
        C │                        │ w4     w5 │
          │                        └───────────┘

    Shard │
          │                     ┌───────────┐
        A │                     │ w2     w3 │
          │                     └/─────────\┘
          │                     /           \
          │        ┌───────────/┐            \
        B │        │ w6      w1 │             \
          │        └/───────────┘              \
          │        /                            \
          │   ┌───/┐                            ┌\───┐
        C │   │ w5 │                            │ w4 │
          │   └────┘                            └────┘

There's no reason to prefer either of these as they have the same _N_ and _D_
values. What's important to note is that the decision to merge [_w1_] and
[_w6_], or to merge [_w4_] and [_w5_] are mutually exclusive; once one of these
merges has been done, the other is illegal without violating causality.
