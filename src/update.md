# `update (DocPath, (Doc) -> Doc)`

Updates to documents are expressed by giving the document's path, and a function
that takes the document's current state and returns its new state, possibly
asynchronously.

```js
db.update('/hello.txt', async (doc) => {
  return { ...doc, modified: true }
})
```

This design makes it possible to deal with write conflicts in CAS adapters
cleanly, by re-reading the shard, extracting the document from it and
re-applying the update function to it. If the document does not yet exist, the
input to the updater function is `null`. If the updater function returns `null`
then the document will be deleted, according to the description of the
`remove()` function below.

We'll deal with normal updates that produce a new value first. A document update
requires at least two item updates:

- Update the document item itself with the new value
- Update the document's parent directory to make sure it includes the document's
  name

We will call the first operation `put()` and the second one `link()`. If a
document has multiple ancestor directories, then multiple `link()` operations
are required in order to list each intermediate directory inside its parent. For
example, updating the doc `/path/to/doc.txt` requires the following item
updates:

- `put('/path/to/doc.txt', newDoc)`
- `link('/path/to/', 'doc.txt')`
- `link('/path/', 'to/')`
- `link('/', 'path/')`

Because directory names end with a slash and document names do not, it is legal
for documents and directories to have the same textual name, which is normally
not allowed in filesystems.

Performing any one of these operations requires reading the relevant shard into
memory, making some modification to it, and then writing it back to storage. As
we know all the affected items at the beginning of the operation, the most
obvious way to optimise this operation is to load all the required shards in
parallel, make any required changes to them, and then write them all back in
parallel.

For example, to update `/my/note` we need to perform three operations:
`put('/my/note', newDoc)`, `link('/my/', 'note')` and `link('/', 'my/')`, so we
need to interact with up to three shards. Let's say we do all the work in
parallel; each `WRITE` operation is labelled with the changes it contains
relative to the previous state:

    Shard │
          │   ┌─────────────────┐         ┌─────────────────┐
        A │   │ READ ░░░░░░░░░░░│         │ WRITE ░░░░░░░░░░│
          │   ├─────────────────┘         ├─────────────────┘
          │   │                           │
          │   │ ┌─────────────────┐       │ ┌─────────────────┐
        B │   │ │ READ ░░░░░░░░░░░│       │ │ WRITE ░░░░░░░░░░│
          │   │ ├─────────────────┘       │ ├─────────────────┘
          │   │ │                         │ │
          │   │ │ ┌─────────────────┐     │ │ ┌─────────────────┐
        C │   │ │ │ READ ░░░░░░░░░░░│     │ │ │ WRITE ░░░░░░░░░░│
          │   │ │ ├─────────────────┘     │ │ ├─────────────────┘
              │ │ │                       │ │ │
              │ │ └─ list('/')            │ │ └─ link('/', 'my/')
              │ │                         │ │
              │ └─ list('/my/')           │ └─ link('/my/', 'note')
              │                           │
              └─ get('/my/note')          └─ put('/my/note')

Let's consider how this works against a CAS adapter. If any shard write fails
with a conflict, then we need to re-read it, re-apply the required changes, and
try the write again. The window for conflicts here should be short as we're
loading all the data we need, making some fast in-memory changes to it, and then
writing everything back as soon as possible. The main source of latency is the
read/write I/O itself.

This execution strategy is fine if we can guarantee that all writes succeed.
Although the shards may briefly be inconsistent, they end up in a state where
the directory listings exactly match the documents that exist. Now imagine that
one of the writes for a `link()` operation fails:

    Shard │
          │   ┌─────────────────┐         ┌─────────────────┐
        A │   │ READ ░░░░░░░░░░░│         │ WRITE ░░░░░░░░░░│
          │   ├─────────────────┘         ├─────────────────┘
          │   │                           │
          │   │ ┌─────────────────┐       │ ┌────────────── ⋯
        B │   │ │ READ ░░░░░░░░░░░│       │ │ WRITE ░░░░░░░ ⋯
          │   │ ├─────────────────┘       │ ├────────────── ⋯
          │   │ │                         │ │
          │   │ │ ┌─────────────────┐     │ │ ┌─────────────────┐
        C │   │ │ │ READ ░░░░░░░░░░░│     │ │ │ WRITE ░░░░░░░░░░│
          │   │ │ ├─────────────────┘     │ │ ├─────────────────┘
              │ │ │                       │ │ │
              │ │ └─ list('/')            │ │ └─ link('/', 'my/')
              │ │                         │ │
              │ └─ list('/my/')           │ └─ link('/my/', 'note') [FAILED]
              │                           │
              └─ get('/my/note')          └─ put('/my/note')

We end up with a state where `/my/note` exists, but the directory item `/my/`
does not contain `note`. This makes `/my/note` undiscoverable by the `list()`
and `find()` functions, which means a recursive delete of the `/my/` or `/`
directory will not remove `/my/note` unless we implement it with a full scan of
all shards.

Earlier we remarked that it's possible in general for `list()` to return
document items that don't exist when we try to read them, or for `list()` to
omit documents that would exist on the next read, because for CAS backends we
cannot guarantee cross-shard consistency. As _transient_ states these aren't a
problem; a temporary inconsistency is to be expected as long as the store is
eventually in a consistent state and we handle write conflicts correctly. But as
a _persistent_ state we see that it's worse for `list()` to omit documents that
do in fact exist, because that will stop `prune()` from working. This gives us a
safety condition for consistent updating of shard files:

> If a document item exists, then its ancestor directory items must contain a
> chain of links that resolve to the document.

Therefore, we should delay the write for the `put()` operation until all the
`link()` operations have been successfully written, producing the following
sequence. We have given each write a number, and in parentheses indicated which
writes each is dependent on, if any.

    Shard │
          │   ┌─────────────────┐                               ┌───────────────────┐
        A │   │ READ ░░░░░░░░░░░│                               │ WRITE 3 (1, 2) ░░░│
          │   ├─────────────────┘                               ├───────────────────┘
          │   │                                                 │
          │   │ ┌─────────────────┐     ┌─────────────────┐     └─ put('/my/note')
        B │   │ │ READ ░░░░░░░░░░░│     │ WRITE 1 ░░░░░░░░│
          │   │ ├─────────────────┘     ├─────────────────┘
          │   │ │                       │
          │   │ │ ┌─────────────────┐   │ ┌─────────────────┐
        C │   │ │ │ READ ░░░░░░░░░░░│   │ │ WRITE 2 ░░░░░░░░│
          │   │ │ ├─────────────────┘   │ ├─────────────────┘
              │ │ │                     │ │
              │ │ └─ list('/')          │ └─ link('/', 'my/')
              │ │                       │
              │ └─ list('/my/')         └─ link('/my/', 'note')
              │
              └─ get('/my/note')

Note that the mutual ordering of the `link()` operations is not important; the
important thing is that we don't create _documents_ until all their parent
directories are correct and the document can therefore be returned by `find()`
correctly. This implicitly means we might apply some `link()` changes and then
fail to apply a `put()`, so it's possible that directories contain items that do
not in fact exist. It should be possible to detect and repair this scenario,
which we'll examine in more detail when we discuss `prune()`.

So the general structure of `update()` operations is a two-level dependency
graph where a `put()` operation depends on one or more `link()` operations.

    ┌───────────────────────────────┐
    │ link('/', 'path/')            │────.
    └───────────────────────────────┘     \
                                           \
    ┌───────────────────────────────┐       \┌──────────────────────────────┐
    │ link('/path/', 'to/')         │────────│ put('/path/to/doc.txt', doc) │
    └───────────────────────────────┘       /└──────────────────────────────┘
                                           /
    ┌───────────────────────────────┐     /
    │ link('/path/to/', 'doc.txt')  │────'
    └───────────────────────────────┘

In VaultDB's data model, all updates will depend on a `link()` operation on the
root directory. To avoid redundant work and reduce the chance of conflicts, we
may be able to avoid performing a write for a `link()` if it already contains
the correct items. For example, if `/my/note` itself is a new document, but the
`/my/` directly already contains other items and so already appears in `/`, we
can remove one write from the sequence:

    Shard │
          │   ┌─────────────────┐                             ┌───────────────────┐
        A │   │ READ ░░░░░░░░░░░│                             │ WRITE 3 (1, 2) ░░░│
          │   ├─────────────────┘                             ├───────────────────┘
          │   │                                               │
          │   │ ┌─────────────────┐     ┌─────────────────┐   └─ put('/my/note')
        B │   │ │ READ ░░░░░░░░░░░│     │ WRITE 1 ░░░░░░░░│
          │   │ ├─────────────────┘     ├─────────────────┘
          │   │ │                       │
          │   │ │ ┌─────────────────┐   │ ┌─┐
        C │   │ │ │ READ ░░░░░░░░░░░│   │ │░│ WRITE 2 (SKIPPED)
          │   │ │ ├─────────────────┘   │ ├─┘
              │ │ │                     │ │
              │ │ └─ list('/')          │ └─ link('/', 'my/')
              │ │                       │
              │ └─ list('/my/')         └─ link('/my/', 'note')
              │
              └─ get('/my/note')

We still need to _read_ the shard for the `/` directory to check its state, but
if the shard is unmodified by our changes, we may not need to write it again.

Another situation in which we can perform fewer writes is where two operations
target the same shard. For example, if both directory items reside in the same
shard, we can apply both `link()` operations and perform a single write.

    Shard │
          │   ┌─────────────────┐                               ┌───────────────────┐
        A │   │ READ ░░░░░░░░░░░│                               │ WRITE 2 (1) ░░░░░░│
          │   ├─────────────────┘                               ├───────────────────┘
          │   │                                                 │
          │   │ ┌─────────────────┐     ┌─────────────────┐     └─ put('/my/note')
        B │   │ │ READ ░░░░░░░░░░░│     │ WRITE 1 ░░░░░░░░│
          │   │ ├─────────────────┘     ├─────────────────┘
              │ │                       │
              │ ├─ list('/')            ├─ link('/', 'my/')
              │ └─ list('/my/')         └─ link('/my/', 'note')
              │
              └─ get('/my/note')

If one of the `link()` operations targets the same shard as the `put()`, we may
be tempted to commit the `link()` to storage first, and then do another write to
commit the `put()`:

    Shard │
          │   ┌─────────────────┐     ┌─────────────────────┐   ┌───────────────────┐
        A │   │ READ ░░░░░░░░░░░│     │ WRITE 1 ░░░░░░░░░░░░│   │ WRITE 3 (1, 2) ░░░│
          │   ├─────────────────┘     ├─────────────────────┘   ├───────────────────┘
          │   │                       │                         │
          │   │ ┌─────────────────┐   │ ┌─────────────────┐     └─ put('/my/note')
        B │   │ │ READ ░░░░░░░░░░░│   │ │ WRITE 2 ░░░░░░░░│
          │   │ ├─────────────────┘   │ ├─────────────────┘
              │ │                     │ │
              │ └─ list('/')          │ └─ link('/', 'my/')
              │                       │
              ├─ list('/my/')         └─ link('/my/', 'note')
              └─ get('/my/note')

However, this is not necessary. The dependency constraint is that the `put()`
must not happen before all the `link()` operations have been committed. This
doesn't forbid us from putting some of the `link()` calls in the same write as
the `put()`, as long as this write is performed after all others are complete.
This write either succeeds, and the `put()` does not precede any `link()` calls,
or it fails and the `put()` doesn't happen at all. Therefore we can defer the
`link()` into the same write as the `put()`.

    Shard │
          │   ┌─────────────────┐                           ┌───────────────────┐
        A │   │ READ ░░░░░░░░░░░│                           │ WRITE 2 (1) ░░░░░░│
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

Although `update()` calls in VaultDB only have a two-level dependency graph, we
can generalise some rules about how the Shard Manager should schedule writes.
When one change _X_ depends on another change _Y_, we should only apply _X_ and
write it after a write containing change _Y_ has successfully committed. A chain
of such dependencies results in writes being performed sequentially:

    Shard │
          │   ┌─────────────────┐   ┌───────────────┐
        A │   │ READ ░░░░░░░░░░░│   │ WRITE 1 ░░░░░░│
          │   └─────────────────┘   ├───────────────┘
          │                         └─ Z             \
          │   ┌─────────────────┐                     ┌───────────────┐
        B │   │ READ ░░░░░░░░░░░│                     │ WRITE 2 (1) ░░│
          │   └─────────────────┘                     ├───────────────┘
          │                                           └─ Y             \
          │   ┌─────────────────┐                                       ┌───────────────┐
        C │   │ READ ░░░░░░░░░░░│                                       │ WRITE 3 (2) ░░│
          │   └─────────────────┘                                       ├───────────────┘
                                                                        └─ X

If two of these changes target the same shard, then we must perform two writes
to that shard rather than merge changes into the same write, otherwise we risk
committing a change before one of its dependencies.

    Shard │
          │   ┌─────────────────┐   ┌───────────────┐                   ┌───────────────┐
        A │   │ READ ░░░░░░░░░░░│   │ WRITE 1 ░░░░░░│                   │ WRITE 3 (2) ░░│
          │   └─────────────────┘   ├───────────────┘                   └─┬─────────────┘
          │                         └─ Z             \                 /  └─ X
          │   ┌─────────────────┐                     ┌───────────────┐
        B │   │ READ ░░░░░░░░░░░│                     │ WRITE 2 (1) ░░│
          │   └─────────────────┘                     ├───────────────┘
                                                      └─ Y

But if change _Y_ does not depend on _Z_, then _Z_ can be deferred into the same
write as _X_, which happens after _Y_ is committed:

    Shard │
          │   ┌─────────────────┐                       ┌─────────────────┐
        A │   │ READ ░░░░░░░░░░░│                       │ WRITE 2 (1) ░░░░│
          │   └─────────────────┘                       └─┬───────────────┘
          │                                            /  ├─ X
          │   ┌─────────────────┐   ┌─────────────────┐   └─ Z
        B │   │ READ ░░░░░░░░░░░│   │ WRITE 1 ░░░░░░░░│
          │   └─────────────────┘   ├─────────────────┘
                                    └─ Y

This intuition lets us plan an algorithm for grouping logical changes into
writes to the storage adapter.

- Expand the high-level operation into a set of individual item operations. For
  example, `update('/my/note')` expands into the lower level item operations
  `link('/', 'my/')`, `link('/my/', 'note')` and `put('/my/note', newDoc)`.

- Build an execution plan by submitting each item operation to the Shard
  Manager, giving the name of the shard it targets, its dependencies, and a
  function that transforms the shard from the old to the new state. The `op()`
  function returns a unique identifier that can be used to express dependencies.
  For example, our `update()` operation turns into the following planning code:

  ```js
  let plan = shards.plan()

  let links = [
    plan.op('B', [], (shard) => shard.link('/', 'my/')),
    plan.op('C', [], (shard) => shard.link('/my/', 'note'))
  ]

  let put = plan.op('A', links, (shard) => {
    return shard.put('/my/note', newDoc)
  })
  ```

- The job of the `plan.op()` function is to build a graph of the requested
  operations and group them into writes. A full description of this
  graph-building operation is given below under "Operation scheduling". For each
  shard, it maintains a list of _write groups_. When a new operation is received
  for a shard, we try to add the operation to an existing group, or start a new
  group if the operation cannot be merged into an existing group. An operation
  may join a group as long as it does not depend on any of the operations in
  that group via a write group on another shard. In our earlier example,
  operation _X_ cannot be grouped with _Z_, because _X_ depends on _Z_ via _Y_
  which targets a different shard.

  ```
  Shard │
        │   ┌─────────────────┐   ┌───────────────┐                   ┌───────────────┐
      A │   │ READ ░░░░░░░░░░░│   │ WRITE 1 ░░░░░░│                   │ WRITE 3 (2) ░░│
        │   └─────────────────┘   ├───────────────┘                   └─┬─────────────┘
        │                         └─ Z             \                 /  └─ X
        │   ┌─────────────────┐                     ┌───────────────┐
      B │   │ READ ░░░░░░░░░░░│                     │ WRITE 2 (1) ░░│
        │   └─────────────────┘                     ├───────────────┘
                                                    └─ Y
  ```

  An operation that depends _directly_ on another can be merged into the same
  group, as we saw in our example of a `put()` and `link()` operation in the
  same write. The graphs for `update()` calls do not contain indirect
  dependencies (`put()` depends on `link()`, which depends on nothing), but they
  can form _crossed dependencies_ which prevent all writes merging into a single
  group, for example:

  - Shard _A_ contains directory item `/alice/` and document item `/bob/doc`
  - Shard _B_ contains directory item `/bob/` and document item `/alice/doc`

  If we want to update `/alice/doc` and `/bob/doc` in the same task, we must
  perform two writes to one of the shards, otherwise we risk writing a document
  before its parent directory is up to date. This example is worked through
  under "Operation scheduling" below.

- Once the graph is completed, the writes can be executed as follows:
  - Begin by initiating a read for all the affected shards. Further work on each
    shard can begin as soon as its data is loaded into memory.
  - For every operation in the graph:
    - Wait for its dependencies to complete.
    - Initiate a write for the operation's group if one is not already running.
  - Performing a write for a group means:
    - Take the state from the last successful read or write for the relevant
      shard.
    - Apply the update functions of all the operations in the group to get the
      new state.
    - If the shard's content has changed, write it back to storage. Otherwise,
      immediately mark all its operations as complete.
    - If the write fails due to a conflict, we need to restart the operation.
      This will be examined in more detail when we discuss `remove()` and
      operation scheduling.
    - If the write succeeds, mark all operations in the group as complete.

The Shard Manager should make sure that only one write operation is in process
for any shard at a time. Once complete, the state of that write, including the
resulting version ID, becomes the input state for further write groups. We do
not need to re-read a shard before each write -- we read the shard once at the
beginning to get its initial state, and then incrementally update that state
with each write. We only need to re-read a shard if a write produces a conflict,
indicating that another process has modified it concurrently.

The design of our write scheduling algorithm has to assume that writes to
separate files are independent, because it has to work with CAS adapters that
aren't able to provide multi-file transactions. There is therefore little
advantage to designing a different algorithm for Locking adapters, which would
predominantly be used to access data on local disk or in memory, and would
benefit much less from any performance optimisations than adapters that work
over the network.

For operations like `update()` where all the required shards are known in
advance, a Locking implementation is straightforward: we acquire locks on all
required shards, in name order to avoid deadlocks, perform the necessary
changes, and then release the locks. For operations like `prune()` that
dynamically decide a set of items to remove, we don't know which shards will be
needed in advance and so cannot pre-emptively lock them before commencing work.
A simpler solution will be to implement the CAS interface on top of the Locking
one, and then use the same write scheduler for both systems to ensure
correctness.
