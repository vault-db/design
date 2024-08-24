# `find (DirPath) -> [Path]`

Get a list of all the document paths that are found inside a directory
recursively, not including directories as distinct items, e.g.:

```js
find('/bob/')
// -> ['/bob/pictures/avatar.jpg', '/bob/pictures/header.png']
```

This operation will often be accompanied by fetching the value of every found
document, for example to prepare a bulk export, or displaying all of the
user's current TOTP codes.

Again, `find()` can be optimised using the same strategy as `get()` and
`list()`: we execute it inside a task that caches shard reads. There is one
additional consideration here which is that the reads we need to perform are
necessarily causally related: we `list()` a directory's children, and then
`list()` _their_ children, and so on until we read the end of the tree. The
children can be fetched in parallel, but they can only be fetched _after_ the
`list()` of their parent directory completes.

```js
db.task(async (task) => {
  let root = await task.list('/')
  let children = await Promise.all(root.map((child) => task.list(child)))
})
```

This produces a waterfall step for each directory traversed sequentially, but
any items that reside in the same shard as the first directory can reuse the
cached shard read.

    Shard │
          │                       ┌─────────────────┐
        A │                       │ READ ░░░░░░░░░░░│
          │                       ├─────────────────┘
          │   ┌─────────────────┐ │ ┌─┐
        B │   │ READ ░░░░░░░░░░░│ │ │░│ CACHE
          │   ├─────────────────┘ │ ├─┘
          │   │                   │ │ ┌─────────────────┐
        C │   │                   │ │ │ READ ░░░░░░░░░░░│
          │   │                   │ │ ├─────────────────┘
              │                   │ │ │
              └ list('/')         │ │ └ list('/carol/')
                                  │ │
                                  │ └ list('/bob/')
                                  │
                                  └ list('/alice/')

If we expect that all the shards will be read, for example if we're enumerating
a directory with many descendants, then an explicit signal to load all the
shards at the beginning would be beneficial for reducing the execution time.

If we also want the values of all the documents, then we can perform the
required `get()` calls inside the same task and take advantage of any shards
cached while reading the tree. This also reduces the likelihood of observing
changes applied while the tree was being read, for example the `find()` call
returning a document path that then returns `null` because it was deleted while
the `find()` call was running. However, this may still occur and applications
should take account of that.

Finally, we'll note that if `list()` and `task()` are exposed by the library,
then `find()` can be implemented on top of those without further access to
internal machinery. However it's still useful to provide this function as a
convenience.
