# `get (DocPath) -> Doc`

Read the document at a single path, which might be `null`, e.g.:

```js
get('/alice/notes.txt')
// -> 'contents of notes file'

get('/alice/no.txt')
// -> null

get('/alice/')
// -> ERROR: not a document path
```

The `get()` function requires a single shard read: we hash the path to determine
the shard it resides in, then read that shard. If multiple `get()` calls are
executed close together, it would be beneficial to cache the shard reads.
Although VaultDB instances will usually be short-lived, we should have a way of
scoping the duration of the cache rather than caching shard reads indefinitely.

VaultDB cannot support multi-object transactions in the SQL sense, but we do
need a concept of batches of operations that are planned and executed together.
We will call such a set of related operations a _task_.

The primary purpose of the cache in this case is to prevent reloading the same
shard over and over to read multiple items that reside in it, in the case where
the reads are close together in time. If the user wants indefinite caching,
there should be a way to explicitly indicate this by starting a task and
performing the reads from within that task. Caches should be scoped to that
task, rather than to the whole VaultDB instance.

There's not a strong reason to perform any explicit concurrency control for this
operation. For Locking adapters, it is tempting to pre-emptively lock a set of
shards before reading them, to prevent another process changing the shards
during a bulk read and achieve a form of snapshot isolation. To prevent
deadlock, the shards should be locked in order of their ID. However, this
restricts the ability for multiple processes to read the shards concurrently,
which will be safe the majority of the time.

In any case, for CAS adapters, there is no way we can pre-emptively lock
multiple objects, and given our previous remarks about biasing towards CAS
implementations, it makes sense to avoid guaranteeing any multi-object read
consistency.

If the caller believes that it will be requesting enough documents that it will
become necessary to load all the shards, it may be beneficial to signal this so
that the Shard Manager pre-emptively loads all shards in parallel at the start
of the task, rather than loading each shard on the first read to one of its
items. In some cases, this will reduce the total execution time by parallelising
requests.

For example, say we execute this set of reads:

```js
await db.get('/foo')
await db.get('/bar')
await db.get('/qux')
```

If these items reside in different shards, then we get a waterfall pattern if
each shard is loaded when we first request an item from it:

    Shard │
          │   ┌─────────────────┐
        A │   │ READ ░░░░░░░░░░░│
          │   ├─────────────────┘
          │   │                   ┌─────────────────┐
        B │   │                   │ READ ░░░░░░░░░░░│
          │   │                   ├─────────────────┘
          │   │                   │                   ┌─────────────────┐
        C │   │                   │                   │ READ ░░░░░░░░░░░│
          │   │                   │                   ├─────────────────┘
              │                   │                   │
              └ get('/foo')       └ get('/bar')       └ get('/qux')

If some of these items reside in the same shard, then executing these reads in a
task with caching can reduce the execution time:

```js
db.task(async (task) => {
  await task.get('/foo')
  await task.get('/bar')
  await task.get('/qux')
})
```

    Shard │
          │   ┌─────────────────┐                     ┌─┐
        A │   │ READ ░░░░░░░░░░░│                     │░│ CACHE
          │   ├─────────────────┘                     ├─┘
          │   │                   ┌─────────────────┐ │
        B │   │                   │ READ ░░░░░░░░░░░│ │
          │   │                   ├─────────────────┘ │
              │                   │                   │
              └ get('/foo')       └ get('/bar')       └ get('/qux')

An item `get()` that's initiated while its target shard is being loaded into
cache should wait for that cache load to complete, rather than issuing a fresh
request for the same shard.

Signalling that we want to load all shards at the beginning of the task reduces
the execution time by parallelising shard reads:

```js
db.task(async (task) => {
  await task.preloadShards()
  await task.get('/foo')
  await task.get('/bar')
  await task.get('/qux')
})
```

    Shard │
          │   ┌───────────────────┐ ┌─┐
        A │   │ READ ░░░░░░░░░░░░░│ │░│ CACHE
          │   ├───────────────────┘ ├─┘
          │   │                     │
          │   ├───────────────┐     │ ┌─┐
        B │   │ READ ░░░░░░░░░│     │ │░│ CACHE
          │   ├───────────────┘     │ ├─┘
          │   │                     │ │
          │   ├─────────────────┐   │ │ ┌─┐
        C │   │ READ ░░░░░░░░░░░│   │ │ │░│ CACHE
          │   ├─────────────────┘   │ │ ├─┘
              │                     │ │ │
              └ preloadShards()     │ │ │
                                    │ │ └ get('/qux')
                                    │ │
                                    │ └ get('/bar')
                                    │
                                    └ get('/foo')

The main risk here is that we make many more requests than needed, for example
if we actually only need a minority of the shards or if the database is sparsely
populated. A better way of achieving this would be execute the `get()` calls in
parallel, since they're causally independent:

```js
db.task(async (task) => {
  let reads = ['/foo', '/bar', '/qux'].map((path) => task.get(path))
  let docs = await Promise.all(reads)
})
```

This produces the same access pattern as the previous example, as all shard
reads are started at the same time. If multiple items reside in the same shard,
we should make sure each shard is only read once by making items wait on the
existing shard read if one is already executing. This implementation also avoids
loading shards that don't contain any of the requested items.

This is possibly less intuitive than the sequential `get()` calls in the prior
examples, but is not unusual for JS that wants to parallelise I/O. Either way,
some explicit effort is required to make the database perform reads in advance
of when they're needed.

Also appealing about this API is the fact that a bulk get is not a separate
operation; instead we get optimised loading by wrapping normal `get()` calls in
a task that implicitly caches shards. By having separate `task()` and `get()`
functions rather than a combined `bulkGet()`, we let the application control the
duration of the cache, potentially reusing it across multiple user actions.
