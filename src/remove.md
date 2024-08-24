# `remove (DirPath)`

The `remove()` operation is in some ways the inverse of `update()`. We've seen
how an `update()` is composed of two lower-level operations, `put()` and
`link()`, and that `put()` must be performed only after all the `link()`
operations have succeeded. A typical `update()` looks like this, including the
required `get()` and `list()` to get the current state of the document and its
parent directory:

    Shard │
          │   ┌─────────────┐                       ┌─────────────────────┐
        A │   │ get('/doc') ------------------------- put('/doc', newDoc) │
          │   └─────────────┘                       └/────────────────────┘
          │                                         /
          │                                        /
          │   ┌───────────┐     ┌─────────────────/┐
        B │   │ list('/') ------- link('/', 'doc') │
          │   └───────────┘     └──────────────────┘

This graph permits the four operations to execute in one of three orders; the
`list()`, `link()` and `put()` operations must happen in that order, while the
`get()` can happen at any time before the `put()`.

    Order 1                 Order 2                 Order 3
    ------------------------------------------------------------------------
    list('/')               list('/')               get('/doc')
    link('/', 'doc')        get('/doc')             list('/')
    get('/doc')             link('/', 'doc')        link('/', 'doc')
    put('/doc', newDoc)     put('/doc', newDoc)     put('/doc', newDoc)

`remove()` similarly breaks down into two lower-level operations: `rm()` removes
the document itself, and `unlink()` removes an item from a directory. If
removing the document from its parent directory leaves the directory empty, then
that directly should itself be removed, and so on up the tree. This can be
implemented by getting the `list()` of all the ancestor directories and
performing `unlink()` for those that contain a single name forming a chain
upward from the removed document.

In order to obey the safety condition of a document always being reachable via
links in all its ancestor directories, `remove()` must not perform any
`unlink()` calls before the `rm()` has completed. For now we'll assume that
multiple `unlink()` calls may happen in parallel, but they must not precede the
`rm()`. This constraint gives us the following graph:

    Shard │
          │   ┌─────────────┐     ┌────────────┐
        A │   │ get('/doc') ------- rm('/doc') │
          │   └─────────────┘     └───────────\┘
          │                                    \
          │                                     \
          │   ┌───────────┐                     ┌\───────────────────┐
        B │   │ list('/') ----------------------- unlink('/', 'doc') │
          │   └───────────┘                     └────────────────────┘

(The `get()` here represents loading the shard containing `/doc`; we don't need
the current state of `/doc` itself in order to delete it, but we do need to load
the shard it resides in, in order to modify it.)

These operations can also be performed in one of three orders, which differ only
in the position of the `list()` call relative to other operations:

    Order 1                 Order 2                 Order 3
    ------------------------------------------------------------------------
    get('/doc')             get('/doc')             list('/')
    rm('/doc')              list('/')               get('/doc')
    list('/')               rm('/doc')              rm('/doc')
    unlink('/', 'doc')      unlink('/', 'doc')      unlink('/', 'doc')

If the document has more than one ancestor directory, for example
`/path/to/note.txt`, that doesn't fundamentally change the structure of these
graphs, it just adds more `link()` and `unlink()` calls that can happen in
parallel.

The key risk in implementing `remove()` is that it can be executed concurrently
with an `update()` that affects the same directories, causing a race condition.
For example, if we have a document named `/path/to/note.txt`, then an `update()`
call will call `link('/path/', 'to/')` while a `remove()` may attempt
`unlink('/path/', 'to/')` if it believes the `/path/to/` directory will be left
empty. If `remove('/path/to/note.txt')` is called concurrently with
`update('/path/to/readme.md')`, then `remove()` might try to delete the
`/path/to/`, `/path/` and `/` directories entirely, believing it's deleting the
only document inside them, not knowing that another document is in the process
of being written.

We must therefore pay careful attention to how the operations in `update()` and
`remove()` are executed. In a CAS system we can rely on the underlying adapter
to raise an update conflict if two processes concurrently modify the same
directory item; for example this protects us from removing directories we
believe will become empty if some other process modified them since we called
`list()` to get their current state. The question for us here is how we handle
such conflicts to make sure that databases always remain in a safe state, given
any possible interleaving of the operations inside an `update()` and a
`remove()`.

In general a CAS adapter will raise a conflict in the following cases:

- Between a `get()` call to read a document, and a `put()` or `rm()` call to
  update that document, another process updates the shard that document resides
  in.

- Between a `list()` call to read a directory, and a `link()` or `unlink()` call
  to update that directory, another process updates the shard that directory
  resides in.

This means a CAS adapter will definitely raise a conflict if two processes
concurrently modify the same item, but it will also raise a conflict if any
other item in the same shard is updated at the same time. To detect whether a
conflict is actually due to _our_ item being updated, rather than some other
item in the same shard, it may be desirable to give individual items version
IDs.

We'll say that a document write that comes between a `get()` and a
`put()`/`rm()` for the same document _invalidates_ the `get()`. That is, it
renders the version ID returned by `get()` stale, and the following
`put()`/`rm()` will fail with a conflict error. Similarly, a write to a
directory item can invalidate a `list()`, causing a future `link()`/`unlink()`
to fail. We need to arrange things so that any sequence of operations executed
by a set of VaultDB clients does not leave the data in an unsafe state. Any
operation that would lead to such a state should cause an invalidation
somewhere, leading to unsafe operations being rejected with a conflict.

We can see that Order 1 for the `update()` and `remove()` functions given above
is fundamentally unsafe, as follows. If an `update()` and `remove()` for `/doc`
happen one after the other, it gives the following sequence of operations:

| `update`                          | `remove`
| --------------------------------- | ---------------------------------
| `list('/')`                       |
| `link('/', 'doc')`                |
| `get('/doc')`                     |
| `put('/doc', newDoc)`             |
|                                   | `get('/doc')`
|                                   | `rm('/doc')`
|                                   | `list('/')`
|                                   | `unlink('/', 'doc')`

If `update` and `remove` are executed independently by different clients, then
the operations within each one remain in the same order but the two sets of
operations can become interleaved. For example, this order of operations is also
possible:

| `update`                          | `remove`
| --------------------------------- | ---------------------------------
| `list('/')`                       |
| `link('/', 'doc')`                |
|                                   | `get('/doc')`
|                                   | `rm('/doc')`
|                                   | `list('/')`
|                                   | `unlink('/', 'doc')`
| `get('/doc')`                     |
| `put('/doc', newDoc)`             |

None of the writes in this sequence fail, because none of the `get()` or
`list()` calls are invalidated by some other write coming between them and the
corresponding `put()`/`rm()` or `link()`/`unlink()`. And so we end up executing
`put()` for the document after we called `unlink()` to remove it from its parent
directory, violating the safety condition.

Thinking about how we'd implement Order 1 using a Locking adapter helps
illustrate why it's not safe. To implement `update` in this order it would be
sufficient to do the following:

- Acquire a lock on the shard holding the `/` directory (shard `B`)
- Read shard `B`
- Write shard `B` to update the `/` directory
- Release the lock on shard `B`
- Acquire a lock on the shard holding the `/doc` document (shard `A`)
- Read shard `A`
- Write shard `A` to update the `/doc` document
- Release the lock on shard `A`

At no point are we holding locks on both shards at the same time, and this is
what allows another process to come in and make changes between our updates to
`/` and `/doc` leading to a safety violation. To fix this we need to force the
extent of these locks to overlap. In a CAS system, we are implicitly holding a
lock from the time we read a shard, to the next time we write it -- our write
succeeds as long as nobody else writes to the shard while we were looking at it.

Order 1 being unsafe means the `get()` in `update` must come before the
`link()`; using the locking analogy, this would mean acquiring the lock on `A`
while we're still holding the lock on `B`. Similarly, the `list()` in `remove`
must come before the `rm()`. This gives us the following dependency graph for
`update`:

    Shard │
          │      ┌─────────────┐                    ┌─────────────────────┐
        A │      │ get('/doc') │                    │ put('/doc', newDoc) │
          │      └────────────\┘                    └/────────────────────┘
          │                    \                    /
          │                     \                  /
          │   ┌───────────┐     ┌\────────────────/┐
        B │   │ list('/') ------- link('/', 'doc') │
          │   └───────────┘     └──────────────────┘

And the following graph for `remove`; these graphs allow Orders 2 and 3 but not
Order 1:

    Shard │
          │   ┌─────────────┐     ┌────────────┐
        A │   │ get('/doc') ------- rm('/doc') │
          │   └─────────────┘     └/──────────\┘
          │                       /            \
          │                      /              \
          │          ┌──────────/┐              ┌\───────────────────┐
        B │          │ list('/') │              │ unlink('/', 'doc') │
          │          └───────────┘              └────────────────────┘

Now let's examine Order 2 and see what further constraints it reveals. Without
interleaving, the Order 2 operation sequence for `update` and `remove` looks
like this:

| `update`                          | `remove`
| --------------------------------- | ---------------------------------
| `list('/')`                       |
| `get('/doc')`                     |
| `link('/', 'doc')`                |
| `put('/doc', newDoc)`             |
|                                   | `get('/doc')`
|                                   | `list('/')`
|                                   | `rm('/doc')`
|                                   | `unlink('/', 'doc')`

To violate safety, we need to find a way to perform a write that produces an
unsafe state, but does not cause a CAS conflict, and so executes successfully.
In our previous example, this happened when `put()` was executed after
`unlink()`:

| `update`                          | `remove`
| --------------------------------- | ---------------------------------
| `list('/')`                       |
| `get('/doc')`                     |
| `link('/', 'doc')`                |
|                                   | `get('/doc')`
|                                   | `list('/')`
|                                   | `rm('/doc')`
|                                   | `unlink('/', 'doc')`
| `put('/doc', newDoc)` ⚠️           |

In this order of operations, `put()` produces a conflict, indicated by ⚠️,
because an `rm()` occurs between the `get()` and `put()` in `update`,
invalidating the `get()`. To avoid this, the `get()` in `update` would have to
happen after the `rm()` operation, for example:

| `update`                          | `remove`
| --------------------------------- | ---------------------------------
| `list('/')`                       |
|                                   | `get('/doc')`
|                                   | `list('/')`
|                                   | `rm('/doc')`
| `get('/doc')`                     |
| `link('/', 'doc')`                |
|                                   | `unlink('/', 'doc')` ⚠️
| `put('/doc', newDoc)`             |

We're not changing the operation order _within_ each function, we're just
changing how the two functions interleave. Placing the `get()` in `update` after
`rm()` forces the `link()` call to happen after the `list()` in `remove`. It
thus invalidates that `list()` call and causes `unlink()` to fail with a
conflict. At no point in this execution is the database in an unsafe state,
because the following three writes are successful:

- `rm()` removes the document but leaves it linked in its parent directory
- `link()` ensures the document is linked
- `put()` writes a new version of the document

The database is never in a state where the document exists, but is not linked,
so this execution is safe. While keeping `get()` after `rm()`, what other
executions are possible? Only two other orderings of the `link()`, `put()` and
`unlink()` calls are possible; either `unlink()` happens first:

| `update`                          | `remove`
| --------------------------------- | ---------------------------------
| `list('/')`                       |
|                                   | `get('/doc')`
|                                   | `list('/')`
|                                   | `rm('/doc')`
| `get('/doc')`                     |
|                                   | `unlink('/', 'doc')`
| `link('/', 'doc')` ⚠️              |
| `put('/doc', newDoc)` ⛔️          |

This invalidates the `list()` in `update` and makes `link()` fail with a
conflict, so we don't actually execute `put()` at all (indicated by ⛔️). This is
equivalent to the `remove` fully completing and the `update` not happening at
all, which is safe. The other possible ordering is that `unlink()` happens last:

| `update`                          | `remove`
| --------------------------------- | ---------------------------------
| `list('/')`                       |
|                                   | `get('/doc')`
|                                   | `list('/')`
|                                   | `rm('/doc')`
| `get('/doc')`                     |
| `link('/', 'doc')`                |
| `put('/doc', newDoc)`             |
|                                   | `unlink('/', 'doc')` ⚠️

This is equivalent to the earlier case where `link()` causes `unlink()` to fail.
`update` completes successfully, and `remove` partially executes but does not
introduce a safety violation.

The above sequence of operations demonstrates two important constraints. First,
it's important that `unlink()` fails here, because if it succeeded it would
unlink the existing `/doc` document, leaving things in an unsafe state. This can
only happen if we execute the `link()` call, even if the `/` directory already
contains `doc`, purely to update the shard's version ID and cause the `unlink()`
call to fail. We cannot skip `link()` calls that do not change a directory's
contents as we'd earlier hoped we might. (If the CAS adapter's versioning is
just a content hash, then writing the same content to it may not change its
version. We can force a version change by re-encrypting the relevant directory
item with new random keys, thereby changing the shard's contents, or by
incrementing a counter in the shard's metadata whenever we write it. This also
prevents the [ABA problem][4] from happening.)

[4]: https://en.wikipedia.org/wiki/ABA_problem

Second, it shows that if an operation fails with a conflict, we cannot retry
just that operation -- we must retry the entire function it was part of. For
example, imagine we handle the above conflicted `unlink()` call by just
re-reading the `/` directory and trying the `unlink` again:

| `update`                          | `remove`
| --------------------------------- | ---------------------------------
| `list('/')`                       |
|                                   | `get('/doc')`
|                                   | `list('/')`
|                                   | `rm('/doc')`
| `get('/doc')`                     |
| `link('/', 'doc')`                |
| `put('/doc', newDoc)`             |
|                                   | `unlink('/', 'doc')` ⚠️
|                                   | `list('/')`
|                                   | `unlink('/', 'doc')`

The second `unlink()` succeeds, but now we've caused a safety violation by
unlinking `/doc` when it still exists. Instead we need to redo any reads
affected by the conflict to (i.e. the `link()` call) to get their new state and
version IDs, and then redo all the writes for the `remove` function. We don't
need to redo reads for shards we've not seen a conflict on, for example, we
don't need to redo the `get()` call here, we can continue to use the version ID
we got from the `rm()` call, as long as there's no evidence that version is now
stale.

| `update`                          | `remove`
| --------------------------------- | ---------------------------------
| `list('/')`                       |
|                                   | `get('/doc')`
|                                   | `list('/')`
|                                   | `rm('/doc')`
| `get('/doc')`                     |
| `link('/', 'doc')`                |
| `put('/doc', newDoc)`             |
|                                   | `unlink('/', 'doc')` ⚠️
|                                   | `list('/')`
|                                   | `rm('/doc')`
|                                   | `unlink('/', 'doc')`

This leaves the database in a safe state because `/doc` is removed before being
unlinked. If any other client updates `/` or `/doc` while this `remove` is
retrying, then we will get further conflicts and we can start over once again.

Now, recall that we're trying to find executions that lead to unsafe states but
don't trigger CAS conflicts. We just did this by looking at traces where `get()`
in `update` is not invalidated because it occurs after `rm()`. The other way to
avoid a conflict on the `/doc` shard would be for `rm()` to happen after
`put()`, that is:

| `update`                          | `remove`
| --------------------------------- | ---------------------------------
|                                   | `get('/doc')`
|                                   | `list('/')`
| `list('/')`                       |
| `get('/doc')`                     |
| `link('/', 'doc')`                |
| `put('/doc', newDoc)`             |
|                                   | `rm('/doc')` ⚠️
|                                   | `unlink('/', 'doc')` ⛔️

In this execution, the `get()` in `remove` is invalidated by the `put()`,
causing `rm()` to fail with a conflict. To avoid this, the `get()` in `remove`
would have to happen after `put()` as well:

| `update`                          | `remove`
| --------------------------------- | ---------------------------------
| `list('/')`                       |
| `get('/doc')`                     |
| `link('/', 'doc')`                |
| `put('/doc', newDoc)`             |
|                                   | `get('/doc')`
|                                   | `list('/')`
|                                   | `rm('/doc')`
|                                   | `unlink('/', 'doc')`

But this is just the two functions executing sequentially, so no race condition
is possible. Each function executes to completion, keeping the database in a
safe state at all times by the way their internal operations are ordered.

Other attempts to construct a safety violation without a conflict end in the
same way. For example, in order for the `unlink()` call not to cause a conflict
in the `update` function, it must happen either before the `list()` call in
`update` (which would be a sequential, non-interleaved execution), or it must
happen after the `link()` call, as in:

| `update`                          | `remove`
| --------------------------------- | ---------------------------------
|                                   | `get('/doc')`
|                                   | `list('/')`
|                                   | `rm('/doc')`
| `list('/')`                       |
| `get('/doc')`                     |
| `link('/', 'doc')`                |
|                                   | `unlink('/', 'doc')` ⚠️
| `put('/doc', newDoc)`             |

Calling `unlink()` here would be unsafe, as it would unlink a doc that is then
created via `put()`. Thankfully, the `unlink()` fails with a conflict because
`link()` invalidates the `list()` in `remove`. To avoid this, the `list()` in
`remove` would have to happen after `link()`:

| `update`                          | `remove`
| --------------------------------- | ---------------------------------
|                                   | `get('/doc')`
| `list('/')`                       |
| `get('/doc')`                     |
| `link('/', 'doc')`                |
|                                   | `list('/')`
|                                   | `rm('/doc')`
|                                   | `unlink('/', 'doc')`
| `put('/doc', newDoc)` ⚠️           |

This forces `rm()` to happen later, which causes `put()` to fail, or `put()`
happens first and causes `rm()` to fail. Any attempt to escape a conflict either
moves the conflict to somewhere else, or produces something equivalent to safe
sequential execution. None of these cases lead to a safety violation without a
conflict arising somewhere.

Similar arguments apply for Order 3 executions. The key thing is that by reading
_all_ the affected shards before performing any writes, and thus getting their
version IDs before we try to change anything, we're essentially acquiring
overlapping locks that will alert us if anything changes from the initial state
while we're updating things. The `update()` and `remove()` operations are each
individually constructed so that we end in a safe state if they are partially
executed, and this additional ordering constraint means any concurrent execution
of one of these functions will receive a conflict if they attempt to create an
unsafe state.

All `remove()` calls will involve at least one `unlink()` operation, to remove
the document from its parent directory. The above examples consider a document
that's a direct child of the root directory `/`. Documents nested further down
the tree will potentially require more than one `unlink()` call, if deleting
them would leave their parent directory empty. For example, consider a database
containing this tree of documents:

    /
    └─┬ path/
      ├── a.txt
      └─┬ to/
        └── b.txt

This tree contains five items:

| path             | type      | value
| ---------------- | --------- | ------------------
| `/`              | directory | `['path/']`
| `/path/`         | directory | `['a.txt', 'to/']`
| `/path/a.txt`    | document  | `<blob>`
| `/path/to/`      | directory | `['b.txt']`
| `/path/to/b.txt` | document  | `<blob>`

If we remove `/path/to/b.txt`, this should implicitly remove the `/path/to/`
directory, because `b.txt` is the only document in that directory. It's not a
safety violation for empty directories to exist, but it reduces the efficiency
of the `find()` and `prune()` operations so in general we'd like to avoid them.

When performing a `remove('/path/to/b.txt')` operation, we will first read all
the relevant items -- the shards containing the `/`, `/path/` and `/path/to/`
directories and the `/path/to/b.txt` document itself. We'll observe that
`/path/to/` only contains a single item, and removing that item would leave it
empty, and so we can unlink it from its parent. In general we can continue doing
this all the way up the directory tree until we find a directory with more than
one item. So, this `remove()` call would perform the following operations:

- `rm('/path/to/b.txt')`
- `unlink('/path/', 'to/')`
- `unlink('/path/to/', 'b.txt')`

These would leave the database in the final state...

    /
    └─┬ path/
      └── a.txt

... containing just three items:

| path          | type      | value
| ------------- | --------- | -----------
| `/`           | directory | `['path/']`
| `/path/`      | directory | `['a.txt']`
| `/path/a.txt` | document  | `<blob>`

We must make sure that we don't delete ancestor directories that are not in fact
empty -- if we remove the `/path/to/` directory but it actually contains items
other than `b.txt`, then we may leave some documents unlinked. Imagine that a
call to `update('/path/to/c.txt')` occurs concurrently with this `remove()`
call, consisting of the following operations:

- `link('/', 'path/')`
- `link('/path/', 'to/')`
- `link('/path/to/', 'c.txt')`
- `put('/path/to/c.txt', doc)`

From the starting state containing `b.txt`, this would produce the following
state...

    /
    └─┬ path/
      ├── a.txt
      └─┬ to/
        ├── b.txt
        └── c.txt

... consisting of six items:

| path             | type      | value
| ---------------- | --------- | --------------------
| `/`              | directory | `['path/']`
| `/path/`         | directory | `['a.txt', 'to/']`
| `/path/a.txt`    | document  | `<blob>`
| `/path/to/`      | directory | `['b.txt', 'c.txt']`
| `/path/to/b.txt` | document  | `<blob>`
| `/path/to/c.txt` | document  | `<blob>`

Is it possible to execute these `remove()` and `update()` calls concurrently in
a way that leads to an unsafe state, by incorrectly deleting the `/path/to/`
directory? If these are executed sequentially, this is the complete set of
operations involved, where we have labelled each operation with a letter to make
them easier to refer to:

|     | `update`                         |     |`remove`
| --- | -------------------------------- | --- | --------------------------------
|     |                                  | _A_ | `list('/')`
|     |                                  | _B_ | `list('/path/')`
|     |                                  | _C_ | `list('/path/to/')`
|     |                                  | _D_ | `get('/path/to/b.txt')`
|     |                                  |     | `----`
|     |                                  | _E_ | `rm('/path/to/b.txt')`
|     |                                  |     | `----`
|     |                                  | _F_ | `unlink('/path/', 'to/')`
|     |                                  | _G_ | `unlink('/path/to/', 'b.txt')`
| _H_ | `list('/')`                      |     |
| _J_ | `list('/path/')`                 |     |
| _K_ | `list('/path/to/')`              |     |
| _L_ | `get('/path/to/c.txt')`          |     |
|     | `----`                           |     |
| _M_ | `link('/', 'path/')`             |     |
| _N_ | `link('/path/', 'to/')`          |     |
| _P_ | `link('/path/to/', 'c.txt')`     |     |
|     | `----`                           |     |
| _Q_ | `put('/path/to/c.txt', doc)`     |     |

The lines reading `----` denote _barriers_: within the `update` and `remove`
calls, operations may be executed in parallel and so run in unpredictable
orders, but the implementation makes sure they're not re-ordered across these
barriers. First each call performs all its reads -- the `list()` and `get()`
calls. `update` then performs a set of `link()` operations, and then a `put()`
if all those succeed. `remove` performs `rm()` and then a set of `unlink()`
calls if that succeeds. `list()` and `get()` calls may be re-ordered relative to
each other, `link()` calls may happen in any order, as may `unlink()` calls. But
`link()` cannot happen after `put()`, nor can `rm()` happen after `unlink()`, in
order to preserve our safety condition.

Given this set of operations, is it possible for them to execute in an order
such that we mistakenly call _F_: `unlink('/path/', 'to/')` and end up leaving
`c.txt` unlinked? In a sequential execution of `remove` followed by `update`,
this trivially does not happen, because `update` does all the necessary `link()`
calls before writing `c.txt`. If the calls overlap in time, then for
`unlink('/path/', 'to/')` to be written incorrectly, a couple of conditions must
hold:

- The call _C_: `list('/path/to/')` must return a list containing a single item,
  so that we decide to perform call _F_. This means _C_ must happen _before_
  call _P_ adds `c.txt` to this directory.

- The call _F_ must succeed without conflict, i.e. no conflicting `link()` call
  happens between the relevant `list()` and `unlink()` calls for `/path/` in
  `remove`. That is, call _N_ must not come between calls _B_ and _F_.

It turns out there is an ordering of these operations allowed by these rules
that produces a problem:

|     | `update`                         |     |`remove`
| --- | -------------------------------- | --- | --------------------------------
| _H_ | `list('/')`                      |     |
| _J_ | `list('/path/')`                 |     |
| _K_ | `list('/path/to/')`              |     |
| _L_ | `get('/path/to/c.txt')`          |     |
|     | `----`                           |     |
| _M_ | `link('/', 'path/')`             |     |
| _N_ | `link('/path/', 'to/')`          |     |
|     |                                  | _A_ | `list('/')`
|     |                                  | _B_ | `list('/path/')`
|     |                                  | _C_ | `list('/path/to/')`
| _P_ | `link('/path/to/', 'c.txt')`     |     |
|     | `----`                           |     |
| _Q_ | `put('/path/to/c.txt', doc)`     |     |
|     |                                  | _D_ | `get('/path/to/b.txt')`
|     |                                  |     | `----`
|     |                                  | _E_ | `rm('/path/to/b.txt')`
|     |                                  |     | `----`
|     |                                  | _F_ | `unlink('/path/', 'to/')`
|     |                                  | _G_ | `unlink('/path/to/', 'b.txt')` ⚠️

As required, _P_ comes after _C_, and _N_ is not between _B_ and _F_. However,
_P_ happens between _C_ and _G_ and causes _G_ to fail with a conflict, as _C_
has been invalidated. The `update` function runs to completion: calls _M_, _N_
and _P_ happen before `remove` performs any writes, and _Q_ affects an item that
`remove` does not touch. `remove` fails with a conflict, but only after _F_ has
run and unlinked the `/path/to/` directory. This means that after _F_ completes,
the document `/path/to/c.txt` exists but is not fully linked since the `/path/`
directory does not contain `to/`.

We can address this by forcing `unlink()` calls to execute sequentially,
starting with the deepest directory. That is, _F_ must happen after _G_, which
we'll denote by adding a barrier between them.

|     | `update`                         |     |`remove`
| --- | -------------------------------- | --- | --------------------------------
| _H_ | `list('/')`                      |     |
| _J_ | `list('/path/')`                 |     |
| _K_ | `list('/path/to/')`              |     |
| _L_ | `get('/path/to/c.txt')`          |     |
|     | `----`                           |     |
| _M_ | `link('/', 'path/')`             |     |
| _N_ | `link('/path/', 'to/')`          |     |
|     |                                  | _A_ | `list('/')`
|     |                                  | _B_ | `list('/path/')`
|     |                                  | _C_ | `list('/path/to/')`
| _P_ | `link('/path/to/', 'c.txt')`     |     |
|     | `----`                           |     |
| _Q_ | `put('/path/to/c.txt', doc)`     |     |
|     |                                  | _D_ | `get('/path/to/b.txt')`
|     |                                  |     | `----`
|     |                                  | _E_ | `rm('/path/to/b.txt')`
|     |                                  |     | `----`
|     |                                  | _G_ | `unlink('/path/to/', 'b.txt')` ⚠️
|     |                                  |     | `----`
|     |                                  | _F_ | `unlink('/path/', 'to/')` ⛔️

This modification to the previous execution means _F_ will no longer execute;
the barrier means that _F_ will only run if _G_ completes successfully. At no
point in this execution do we violate the requirement that all documents are
fully linked to the root directory.

To prevent the conflict on _G_, we would need _P_ not to happen between _C_ and
_G_. Remember we only perform _F_ if _P_ happens after _C_, so the only
situation worth investigating is where _P_ happens after _G_:

|     | `update`                         |     |`remove`
| --- | -------------------------------- | --- | --------------------------------
| _H_ | `list('/')`                      |     |
| _J_ | `list('/path/')`                 |     |
| _K_ | `list('/path/to/')`              |     |
| _L_ | `get('/path/to/c.txt')`          |     |
|     | `----`                           |     |
| _M_ | `link('/', 'path/')`             |     |
| _N_ | `link('/path/', 'to/')`          |     |
|     |                                  | _A_ | `list('/')`
|     |                                  | _B_ | `list('/path/')`
|     |                                  | _C_ | `list('/path/to/')`
|     |                                  | _D_ | `get('/path/to/b.txt')`
|     |                                  |     | `----`
|     |                                  | _E_ | `rm('/path/to/b.txt')`
|     |                                  |     | `----`
|     |                                  | _G_ | `unlink('/path/to/', 'b.txt')`
| _P_ | `link('/path/to/', 'c.txt')` ⚠️   |     |
|     | `----`                           |     |
| _Q_ | `put('/path/to/c.txt', doc)` ⛔️  |     |
|     |                                  |     | `----`
|     |                                  | _F_ | `unlink('/path/', 'to/')`

Now _G_ falls between _K_ and _P_ and causes _P_ to fail with a conflict, and
thus _Q_ does not happen at all. `remove` runs to completion, and `update` fails
to create the document that would have triggered a safety violation.

To avoid this conflict, _K_ would need to happen after _G_. We can move _K_ down
below _L_, but the barrier below that necessitates moving all the other `update`
operations after this after _G_ as well.

|     | `update`                         |     |`remove`
| --- | -------------------------------- | --- | --------------------------------
| _H_ | `list('/')`                      |     |
| _J_ | `list('/path/')`                 |     |
| _L_ | `get('/path/to/c.txt')`          |     |
|     |                                  | _A_ | `list('/')`
|     |                                  | _B_ | `list('/path/')`
|     |                                  | _C_ | `list('/path/to/')`
|     |                                  | _D_ | `get('/path/to/b.txt')`
|     |                                  |     | `----`
|     |                                  | _E_ | `rm('/path/to/b.txt')`
|     |                                  |     | `----`
|     |                                  | _G_ | `unlink('/path/to/', 'b.txt')`
| _K_ | `list('/path/to/')`              |     |
|     | `----`                           |     |
| _M_ | `link('/', 'path/')`             |     |
| _N_ | `link('/path/', 'to/')`          |     |
| _P_ | `link('/path/to/', 'c.txt')`     |     |
|     | `----`                           |     |
| _Q_ | `put('/path/to/c.txt', doc)`     |     |
|     |                                  |     | `----`
|     |                                  | _F_ | `unlink('/path/', 'to/')` ⚠️

Call _N_ now falls between _B_ and _F_ and causes _F_ to fail with a conflict,
and so we don't perform the problematic `unlink()` call and _Q_ is safe. We
can't move _N_ up before _B_ without also moving _K_ before _G_, due to the
barriers, so let's instead try moving _N_ to happen after _F_.

|     | `update`                         |     |`remove`
| --- | -------------------------------- | --- | --------------------------------
| _H_ | `list('/')`                      |     |
| _J_ | `list('/path/')`                 |     |
| _L_ | `get('/path/to/c.txt')`          |     |
|     |                                  | _A_ | `list('/')`
|     |                                  | _B_ | `list('/path/')`
|     |                                  | _C_ | `list('/path/to/')`
|     |                                  | _D_ | `get('/path/to/b.txt')`
|     |                                  |     | `----`
|     |                                  | _E_ | `rm('/path/to/b.txt')`
|     |                                  |     | `----`
|     |                                  | _G_ | `unlink('/path/to/', 'b.txt')`
| _K_ | `list('/path/to/')`              |     |
|     | `----`                           |     |
| _M_ | `link('/', 'path/')`             |     |
| _P_ | `link('/path/to/', 'c.txt')`     |     |
|     |                                  |     | `----`
|     |                                  | _F_ | `unlink('/path/', 'to/')`
| _N_ | `link('/path/', 'to/')` ⚠️        |     |
|     | `----`                           |     |
| _Q_ | `put('/path/to/c.txt', doc)` ⛔️  |     |

_F_ now happens between _J_ and _N_, causing _N_ to fail with a conflict and
therefore _Q_ does not happen at all and we once again have a safe execution.
Avoiding this conflict would either require moving _F_ to happen after _N_,
which is just the preceding example, or moving it before _J_. Even moving _J_ as
late as possible and keeping _K_ after _G_, this gives us:

|     | `update`                         |     |`remove`
| --- | -------------------------------- | --- | --------------------------------
| _H_ | `list('/')`                      |     |
| _L_ | `get('/path/to/c.txt')`          |     |
|     |                                  | _A_ | `list('/')`
|     |                                  | _B_ | `list('/path/')`
|     |                                  | _C_ | `list('/path/to/')`
|     |                                  | _D_ | `get('/path/to/b.txt')`
|     |                                  |     | `----`
|     |                                  | _E_ | `rm('/path/to/b.txt')`
|     |                                  |     | `----`
|     |                                  | _G_ | `unlink('/path/to/', 'b.txt')`
| _K_ | `list('/path/to/')`              |     |
|     |                                  |     | `----`
|     |                                  | _F_ | `unlink('/path/', 'to/')`
| _J_ | `list('/path/')`                 |     |
|     | `----`                           |     |
| _M_ | `link('/', 'path/')`             |     |
| _P_ | `link('/path/to/', 'c.txt')`     |     |
| _N_ | `link('/path/', 'to/')`          |     |
|     | `----`                           |     |
| _Q_ | `put('/path/to/c.txt', doc)`     |     |

This is basically the same as sequential execution. All the writes in `remove`
succeed because `update` hasn't performed any writes yet. `update` then performs
all the required `link()` calls to make the final write _Q_ safe. By trying to
construct an execution in which we end in an unsafe state, and then trying to
avoid the resulting conflicts, we've ended up with a sequential execution.

Let's take an even more extreme example where the entire `update` operation
happens between _G_ and _F_, which themselves cannot be re-ordered. Is it
possible for the call to _G_ to succeed, and still have _F_ happen erroneously?

|     | `update`                         |     |`remove`
| --- | -------------------------------- | --- | --------------------------------
|     |                                  | _A_ | `list('/')`
|     |                                  | _B_ | `list('/path/')`
|     |                                  | _C_ | `list('/path/to/')`
|     |                                  | _D_ | `get('/path/to/b.txt')`
|     |                                  |     | `----`
|     |                                  | _E_ | `rm('/path/to/b.txt')`
|     |                                  |     | `----`
|     |                                  | _G_ | `unlink('/path/to/', 'b.txt')`
| _H_ | `list('/')`                      |     |
| _J_ | `list('/path/')`                 |     |
| _K_ | `list('/path/to/')`              |     |
| _L_ | `get('/path/to/c.txt')`          |     |
|     | `----`                           |     |
| _M_ | `link('/', 'path/')`             |     |
| _N_ | `link('/path/', 'to/')`          |     |
| _P_ | `link('/path/to/', 'c.txt')`     |     |
|     | `----`                           |     |
| _Q_ | `put('/path/to/c.txt', doc)`     |     |
|     |                                  |     | `----`
|     |                                  | _F_ | `unlink('/path/', 'to/')` ⚠️

Here _F_ fails with a conflict because _N_ happens between _B_ and _F_. We can
move _N_ before _B_, but that leaves a conflict on _P_ because _G_ happens
first. If we move _P_ before _G_ then _G_ fails instead, and prevents _F_ from
happening.

|     | `update`                         |     |`remove`
| --- | -------------------------------- | --- | --------------------------------
|     |                                  | _A_ | `list('/')`
|     |                                  | _C_ | `list('/path/to/')`
|     |                                  | _D_ | `get('/path/to/b.txt')`
| _H_ | `list('/')`                      |     |
| _J_ | `list('/path/')`                 |     |
| _K_ | `list('/path/to/')`              |     |
| _L_ | `get('/path/to/c.txt')`          |     |
|     | `----`                           |     |
| _N_ | `link('/path/', 'to/')`          |     |
|     |                                  | _B_ | `list('/path/')`
|     |                                  |     | `----`
|     |                                  | _E_ | `rm('/path/to/b.txt')`
|     |                                  |     | `----`
|     |                                  | _G_ | `unlink('/path/to/', 'b.txt')`
| _M_ | `link('/', 'path/')`             |     |
| _P_ | `link('/path/to/', 'c.txt')` ⚠️   |     |
|     | `----`                           |     |
| _Q_ | `put('/path/to/c.txt', doc)` ⛔️  |     |
|     |                                  |     | `----`
|     |                                  | _F_ | `unlink('/path/', 'to/')`

Or, we can move _N_ after _F_, but that causes a conflict on _N_ that prevents
_Q_ from happening.

|     | `update`                         |     |`remove`
| --- | -------------------------------- | --- | --------------------------------
|     |                                  | _A_ | `list('/')`
|     |                                  | _B_ | `list('/path/')`
|     |                                  | _C_ | `list('/path/to/')`
|     |                                  | _D_ | `get('/path/to/b.txt')`
|     |                                  |     | `----`
|     |                                  | _E_ | `rm('/path/to/b.txt')`
|     |                                  |     | `----`
|     |                                  | _G_ | `unlink('/path/to/', 'b.txt')`
| _H_ | `list('/')`                      |     |
| _J_ | `list('/path/')`                 |     |
| _K_ | `list('/path/to/')`              |     |
| _L_ | `get('/path/to/c.txt')`          |     |
|     | `----`                           |     |
| _M_ | `link('/', 'path/')`             |     |
| _P_ | `link('/path/to/', 'c.txt')`     |     |
|     |                                  |     | `----`
|     |                                  | _F_ | `unlink('/path/', 'to/')`
| _N_ | `link('/path/', 'to/')` ⚠️        |     |
|     | `----`                           |     |
| _Q_ | `put('/path/to/c.txt', doc)` ⛔️  |     |

We see that attempts to order events such that both _F_ and _Q_ execute and
leave the database in an unsafe state ends up producing a conflict somewhere
that prevents this state from arising. The only constraint we've had to add to
the design to achieve this is that `unlink()` calls happen sequentially, in
bottom-up order. Intuitively, this makes sense because `unlink()` is a
_conditional_ operation: we remove a directory from its parent if it becomes
empty as a result of removing its last child. By performing _G_ before _F_, we
confirm that the state of `/path/to/` has not changed since we read it at _C_,
and therefore the decision to perform _F_ is still valid.

Contrast this with `link()` which is unconditional: we always make sure a
document is linked in its parent directories when creating or updating it.
Therefore it makes sense that `unlink()` needs a causal/ordering constraint
placed upon it while `link()` is allowed to execute in parallel. It's
unfortunate that we have to execute `unlink()` calls sequentially because it
will make `remove()` take longer to run. However, we expect `remove()` calls to
happen much less frequently than `update()`, and most `remove()` calls will
require only one `unlink()`, so this is a reasonable trade-off.

Now we know the general logical dependency structure for `remove()`. Whereas
`update()` is structured as a `put()` following a set of concurrent `link()`
calls, `remove()` is an `rm()` call followed by a sequence of one or more
`unlink()` calls for as many parent directories would be left empty by the
removal of the document.

    ┌────────────────┐
    │ rm('/a/b/doc') │
    └────────────────┘
                      \
                       ┌────────────────────────┐
                       │ unlink('/a/b/', 'doc') │
                       └────────────────────────┘
                                                 \
                                                  ┌─────────────────────┐
                                                  │ unlink('/a/', 'b/') │
                                                  └─────────────────────┘

We will briefly consider how these operations should be scheduled for execution
by the Shard Manager. We always need to perform at least one `unlink()` call for
the document's immediate parent directory, so it makes sense to read the shards
for the document itself and its parent directory up front, before executing the
writes.

    Shard │
          │   ┌─────────────┐       ┌─────────────┐
        A │   │ READ ░░░░░░░│       │ WRITE ░░░░░░│
          │   ├─────────────┘       ├─────────────┘
          │   │                     │              \
          │   │ ┌─────────────┐     │               ┌─────────────┐
        B │   │ │ READ ░░░░░░░│     │               │ WRITE ░░░░░░│
          │   │ ├─────────────┘     │               ├─────────────┘
              │ │                   │               │
              │ └─ list('/a/b/')    │               └─ unlink('/a/b/', 'doc')
              │                     │
              └─ get('/a/b/doc')    └─ rm('/a/b/doc')

If further `unlink()` calls are required, we could wait until the first one
succeeds and becomes empty before deciding to perform the second. This would
mean the shard for the grandparent directory is not read until after the first
`unlink()` write is complete.

    Shard │
          │   ┌─────────────┐       ┌─────────────┐
        A │   │ READ ░░░░░░░│       │ WRITE ░░░░░░│
          │   ├─────────────┘       ├─────────────┘
          │   │                     │              \
          │   │ ┌─────────────┐     │               ┌─────────────┐
        B │   │ │ READ ░░░░░░░│     │               │ WRITE ░░░░░░│
          │   │ ├─────────────┘     │               ├─────────────┘
          │   │ │                   │               │              \
          │   │ │                   │               │               ┌─────────────┐   ┌─────────────┐
        C │   │ │                   │               │               │ READ ░░░░░░░│───│ WRITE ░░░░░░│
          │   │ │                   │               │               ├─────────────┘   ├─────────────┘
              │ │                   │               │               │                 │
              │ │                   │               │               └─ list('/a/')    └─ unlink('/a/', 'b/')
              │ │                   │               │
              │ └─ list('/a/b/')    │               └─ unlink('/a/b/', 'doc')
              │                     │
              └─ get('/a/b/doc')    └─ rm('/a/b/doc')

But we don't need to wait for the first `unlink()` call to succeed before making
this decision. We know that if the `/a/b/` directory contains a single item,
then the `/a/b/` directory itself should be removed, assuming its last remaining
child is successfully unlinked. This suggests we should read the shards for all
the ancestor directories up front, and plan all the required writes before the
`rm()` is executed. The dependency chain will make sure that each write is only
executed if the ones before it succeed, so we can make this plan ahead of time
and still avoid performing the second `unlink()` if the first one fails.

    Shard │
          │   ┌─────────────┐       ┌─────────────┐
        A │   │ READ ░░░░░░░│       │ WRITE ░░░░░░│
          │   ├─────────────┘       ├─────────────┘
          │   │                     │              \
          │   │ ┌─────────────┐     │               ┌─────────────┐
        B │   │ │ READ ░░░░░░░│     │               │ WRITE ░░░░░░│
          │   │ ├─────────────┘     │               ├─────────────┘
          │   │ │                   │               │              \
          │   │ │ ┌─────────────┐   │               │               ┌─────────────┐
        C │   │ │ │ READ ░░░░░░░│   │               │               │ WRITE ░░░░░░│
          │   │ │ ├─────────────┘   │               │               ├─────────────┘
              │ │ │                 │               │               │
              │ │ └─ list('/a/')    │               │               └─ unlink('/a/', 'b/')
              │ │                   │               │
              │ └─ list('/a/b/')    │               └─ unlink('/a/b/', 'doc')
              │                     │
              └─ get('/a/b/doc')    └─ rm('/a/b/doc')

This is even more beneficial if the ancestor directories reside in the same
shard. If we wait for the first `unlink()` write to succeed before deciding to
perform the second, then we end up performing two sequential writes to the same
shard.

    Shard │
          │   ┌─────────────┐       ┌─────────────┐
        A │   │ READ ░░░░░░░│       │ WRITE ░░░░░░│
          │   ├─────────────┘       ├─────────────┘
          │   │                     │              \
          │   │ ┌─────────────┐     │               ┌─────────────┐   ┌─────────────┐
        B │   │ │ READ ░░░░░░░│     │               │ WRITE ░░░░░░│───│ WRITE ░░░░░░│
          │   │ ├─────────────┘     │               ├─────────────┘   ├─────────────┘
              │ │                   │               │                 │
              │ │                   │               │                 └─ unlink('/a/', 'b/')
              │ │                   │               │
              │ └─ list('/a/b/')    │               └─ unlink('/a/b/', 'doc')
              │                     │
              └─ get('/a/b/doc')    └─ rm('/a/b/doc')

This is sub-optimal: it's perfectly safe to merge both `unlink()` calls into a
single write. Either they both succeed or they both fail, but importantly the
`unlink()` on the `/a/` directory is only performed if the `/a/b/` one succeeds,
so the causal dependency is maintained and we don't violate our safety
requirement.

    Shard │
          │   ┌─────────────┐       ┌─────────────┐
        A │   │ READ ░░░░░░░│       │ WRITE ░░░░░░│
          │   ├─────────────┘       ├─────────────┘
          │   │                     │              \
          │   │ ┌─────────────┐     │               ┌─────────────┐
        B │   │ │ READ ░░░░░░░│     │               │ WRITE ░░░░░░│
          │   │ ├─────────────┘     │               ├─────────────┘
              │ │                   │               │
              │ ├─ list('/a/')      │               ├─ unlink('/a/', 'b/')
              │ └─ list('/a/b/')    │               └─ unlink('/a/b/', 'doc')
              │                     │
              └─ get('/a/b/doc')    └─ rm('/a/b/doc')

Likewise, the first `unlink()` can be merged into the same write with the
`put()` if both of them target the same shard. Operations that _directly_ depend
on each other can be merged into the same write without violating causality.

The final question to address here is what do to in the case of a conflict being
reported by the backing store. To handle this conflict, we could either:

- Bail and mark the `update()` or `remove()` as failed, or
- Resume the function by restarting it from the beginning, based on the shards'
  new versions.

The first option is safe, because these operations are individually crash-safe
and so halting them part-way through is fine. However the second option isn't
_unsafe_; restarting a `remove()` operation doesn't violate the safety
requirement to make sure all existing documents are linked in their parent
directories.

Rather than a safety violation, this raises a UI question: what should
`remove()` tell the application if it detects a conflict? CAS is so far assumed
to be an internal implementation detail -- we don't expose version information
in the Query API, so a call to `remove()` can be done with no prior read,
meaning the application wants the item deleted no matter what its current state
is. The target use cases we have in mind include command-line credential
managers that expose a "remove item" operation that doesn't ask the user for the
item's current version before proceeding.

While this can be implemented on top of an API that requires an item to be read
before it's deleted, just to get its current version, this isn't how we intend
VaultDB to be used. `update()` hides the internal concurrency control by taking
an update function rather than a new state, and it retries until the update
succeeds. For symmetry it makes sense to treat `remove()` like a special kind of
update and retry it until it succeeds. If the application isn't co-ordinating
the `update()` and `remove()`, it can't have any expectation of them being
applied in any particular order, and once it learns that an operation succeeded,
it can't assume the item continues in that state as some other process might
modify it immediately afterward. So, it doesn't violate any basic assumptions to
have all operations retry until success, possibly with a retry or time limit,
and it improves the usability for the use cases we have in mind.

If we don't expose version information in the API, then there's nothing an
application could sensibly do with a rejected `remove()` other than make the
exact decision we just did about whether to bail or retry based on a blanket
policy, rather than on the state of the document in question. It would only make
sense for VaultDB's Query API to expose rejection as a mode of failure if we
_also_ expose versioning and don't retry operations ourselves, which would
fundamentally change the design.
