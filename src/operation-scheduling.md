# Operation scheduling

High-level write operations in VaultDB perform a set of multiple writes to
individual items inside shard files. For example, `update('/my/note')` breaks
down into the following writes to individual items:

- `put('/my/note')` to update the document item itself
- `link('/my/', 'note')` to ensure `note` is listed in the `/my/` directory
- `link('/', 'my/')` to ensure `my/` is listed in the `/` directory

In general we expect access to the storage to be done over the network, often to
a remote internet server, and therefore to be unreliable and slow. That is,
requests may fail, and they will probably take a long time on average and be
highly variable. A simple `update()` may require only a couple of changes to
items, but more complex operations like a bulk insert or a recursive delete
could require dozens of changes. When multiple operations target the same shard,
it would be desirable to batch them into a smaller number of requests, to reduce
the total time taken to complete an action.

However, we cannot simply group all the operations for each shard into a single
request, as the order in which changes are applied may be important. For
`update()` to be safe, we must only execute the `put()` operation after all the
`link()` operations are completed successfully. Otherwise, if we perform the
`put()` but one of the `link()` requests fails, we create a document that is not
listed in its parent directories, and that stops recursive deletion from
working.

Earlier we saw an example of an optimised `update('/my/note')` call where the
document item and one of the directory items reside in the same shard:

    Shard │
          │   ┌─────────────────┐                           ┌───────────────────┐
        A │   │ READ ░░░░░░░░░░░│                           │ WRITE 2 ░░░░░░░░░░│
          │   ├─────────────────┘                           ├───────────────────┘
          │   │                                             │
          │   │ ┌─────────────────┐   ┌─────────────────┐   ├─ link('/my/', 'note')
        B │   │ │ READ ░░░░░░░░░░░│   │ WRITE 1 ░░░░░░░░│   └─ put('/my/note')
          │   │ ├─────────────────┘   ├─────────────────┘
              │ │                     │
              │ └─ list('/')          └─ link('/', 'my/')
              │
              ├─ list('/my/')
              └─ get('/my/note')

This is safe because write 2 is only executed if write 1 succeeds, so we only
execute the `put()` if all the required `link()` operations are also done. If
write 2 fails, the database is _inconsistent_ in that the `/` directory contains
the item `my/` which may not in fact contain any documents. But it is not
_unsafe_ in that we have not created any documents that are unreachable via
directory lists and therefore cannot be deleted. We've optimised the I/O by
grouping two operations into write 2, but without violating their dependency
order so that the database is left in a safe state at the end of every write. If
writes 1 and 2 were done in parallel, it would be possible for write 1 to fail
but write 2 to succeed, creating an unsafe state.

We have examined the inner workings of the `update()`, `remove()` and `prune()`
functions and determined how the individual changes they make must be ordered to
ensure safety. We would like for the Shard Manager to execute all these
operations, and any others we may add later, using as few network requests as
possible, while making sure that if any request fails, the database is left in a
safe state. Rather than optimising each of the Query functions specifically, we
would like to come up with a general way of optimising a set of writes to the
shards.

In general, given a set of operations and their dependencies, we want to produce
a plan for executing those operations. This planning process will produce a
directed graph of _write groups_, where each group targets a single shard and
contains one or more changes to apply to the shard's data. The writes are
executed in the order dictated by the graph, and writes may be executed in
parallel when no causal dependency exists between them.

These graphs will be characterised by two metrics:

- _N_, the total number of write groups
- _D_, the graph's depth, which is the length of the longest chain of dependent
  groups that must be executed sequentially

We'd like both to be as small as possible but in general it will be hard to
optimise for both at once. It may also be computationally expensive to find the
graph that produces the smallest possible number for one of these metrics.
Instead we'd like an efficient method for constructing a write graph from a set
of operations that will tend to produce _improved_ performance compared to
executing each operation individually in its own write request. We will use some
heuristics based on examples and our knowledge of the likely shape of operation
dependencies in VaultDB.
