# `prune (DirPath)`

The `prune()` function removes all the documents that are stored under the given
directory path. Logically this can be accomplished by using `find()` to get the
names of all such documents, and then calling `remove()` on each one. However,
it will be desirable to optimise this as a distinct operation because we can
plan its execution a little better than a literal set of concurrent `remove()`
calls. These would be likely to conflict with one another, and they would
execute a lot of incremental `unlink()` operations to remove items from
directories one at a time. If we're deleting an entire directory recursively
then we know that we'll want to delete all its internal directories entirely,
rather than incrementally removing their children.

To perform `prune(dir)`, we still perform a `find(dir)` call for the directory
to locate all the documents to remove, and to load the shards containing all the
intermediate directories into the cache. We then plan a set of `rm()` calls for
those documents _and_ any directories inside `dir`. We delete the directory tree
bottom-up starting with the leaf documents; an item can be removed once all its
children are removed. For example, consider the tree we saw earlier:

    /
    └─┬ path/
      ├── a.txt
      └─┬ to/
        ├── b.txt
        └── c.txt

A call to `prune('/path/')` would first call `find('/path/')`, returning the
list `['/path/a.txt', '/path/to/b.txt', '/path/to/c.txt']`. We immediately
schedule an `rm()` for each of these; we'll give each operation an identifying
label:

- _A_: `rm('/path/a.txt')`
- _B_: `rm('/path/to/b.txt')`
- _C_: `rm('/path/to/c.txt')`

There is one internal directory in this tree: `/path/to/`. This can be removed
once all its documents have been removed:

- _D_: `rm('/path/to/')`, depends on _B_ and _C_

Finally `/path/` itself can be removed once its direct children have been
removed:

- _E_: `rm('/path/')`, depends on _A_ and _D_

The removal must be planned bottom-up for the same reason that the `unlink()`
calls in `remove()` must go bottom-up. In a tree containing a single document,
`/path/to/file.txt`, a call to `prune('/path/')` will perform basically the same
operations as `remove('/path/to/file.txt')`, and they'll be prone to the same
race conditions unless we make the removal of a directory dependent on the
removal of its children.

If any of the writes encounters a conflict, the whole operation should restart
from the initial `find()` call. A successful execution of `prune()` should end
with nothing inside the given directory, so if a concurrent `update()` occurs on
a path inside the directory, we must call `find()` again to get an updated view
of the directory's contents. This `find()` call may happen when the directory
has been partially deleted, which is another reason to make sure deletion goes
bottom-up. `prune()` should be considered successful if all its writes after the
initial `find()` complete without conflicts.

Once all the `rm()` calls have been completed, `prune()` can perform the same
check that `remove()` does for empty parent directories. It should at least
`unlink()` the pruned directory from its parent, and may perform further
`unlink()` calls if this leaves the parent directory empty, and so on.

Putting all this together, the full execution of `prune('/path/')` on the above
tree produces the following graph.

                         ┌───────────────────┐
                         │ rm('/path/a.txt') │
                         └──────────────────\┘
                                             \
    ┌──────────────────────┐                 ┌\─────────────┐   ┌──────────────────────┐
    │ rm('/path/to/b.txt') │                 │ rm('/path/') ----- unlink('/', 'path/') │
    └─────────────────────\┘                 └/─────────────┘   └──────────────────────┘
                           \                 /
                           ┌\───────────────/┐
                           │ rm('/path/to/') │
                           └/────────────────┘
                           /
    ┌─────────────────────/┐
    │ rm('/path/to/c.txt') │
    └──────────────────────┘
