# `list (DirPath) -> [Name]`

Get the child item names of a directory path, which is an empty list if the path
does not exist, e.g.:

```js
list('/')
// -> ['alice/', 'bob/', 'carol/']

list('/alice/')
// -> ['notes.txt']

list('/dave/')
// -> []

list('/bob')
// -> ERROR: not a directory path
```

The `list()` function is identical to `get()` at the shard level: we just need
to know which shard holds the given directory item, and read that shard.
Multiple directory reads can be combined in the same way we just described for
document reads, and directory and document reads can be arbitrarily mixed using
this strategy.

The Shard Manager should be ignorant of the internal structure of shard files,
or the meaning of different types of item. It just needs to know which shards
need to be loaded, and provide optimal access to them.
