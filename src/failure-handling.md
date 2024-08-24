# Handling failed requests

These graphs represent a plan for executing a set of requests so that their
effects don't violate any consistency requirements. Each request in the graph is
only executed after all its dependencies have successfully completed. By the
construction of the different write operations it is safe to stop executing the
graph at any point, because it is safe to partially execute any of the writes
they represent.

So, when any request fails, it is safe to simply stop executing the plan at that
point, so that any requests dependending on it do not execute. However, we would
like to handle some kinds of failure so that our execution recovers from the
failure if possible.

The simplest type of failure to handle is one where retrying the request is not
likely to succeed. For example, an authorization failure most likely indicates
that your credentials are not valid, and repeating the request with the same
credentials will not work. This sort of failure should be reported to the
application as an exception so that appropriate action can be taken. VaultDB
itself should not attempt to recover from the failure, it should just stop
executing the current task. All calls to the Query API that are part of the
current task should fail with an exception. It's possible that this means some
operations are partially executed, but we have tried to design things such that
this is safe.

A more complex type of failure is when we don't receive a response, because of a
network failure during the request. If the request did not reach the server,
then from the server's point of view it's like the request never happened, and
so it's safe to retry it. If the request _did_ reach the server but the response
didn't reach the client, then the request may have been processed but the client
has no idea. If it retries the request it may get a conflict from a CAS adapter,
because the previous successful request changed the shard's version ID. The
client should handle this like it would normally handle a conflict, as detailed
below.

This type of failure can be difficult to detect. In some cases an explicit
exception will be raised indicating a network error -- a DNS failure, a failure
to make a TCP connection, a TLS certificate failure, a connection reset, etc. If
this happens then the client can immediately retry. But sometimes this failure
manifests as the request just hanging, never returning a response or raising an
exception. To deal with this, a timeout mechanism should be used so that if we
don't get a response within a certain amount of time, we assume the request
failed. Not knowing why the request failed, or even that it may have succeeded,
we may end up repeating a successful request and thereby generating a CAS
conflict, as we've already discussed.

Network failures can be handled by retrying the same literal request again, and
can therefore be handled inside the HTTP client machinery without the Shard
Manager even knowing they're happening. These failures should not result in an
immediate retry. The client should wait some amount of time before attempting
the request again, possibly making use of browser APIs to detect when the client
is online again. Immediate retries while the client is offline are liable to
drain the device's battery with requests that are not likely to succeed. The
client should employ a back-off strategy so that it waits an increasing amount
of time on repeat failures. It should employ a retry limit so that the request
eventually fails outright after a certain number of attempts or after a time
limit has elapsed.

All the above failures can be dealt with entirely inside the adapter-specific
code that actually interacts with the storage. It either retries the request
transparently, or it throws an exception that should bubble all the way back to
the application. The most complex type of failure we need to deal with is a
_conflict_ in a CAS adapter. This happens when a request is syntactically valid,
is presented with valid credentials, but is rejected because it carries a
version ID that doesn't match the shard's current version, indicating another
process modified the shard since we last read it.

As we discussed under the description of the `remove()` function, a conflict
must result in the entire high-level Query procedure being restarted from the
beginning, not just repeating the most recent item-level change. That means that
writes that have already succeeded may need to be redone, and the writes we have
queued up in the graph might change.

For example, imagine we have the following set of three Query operations to
execute, where each is listed with the item-level changes it entails:

- _O1_: `update('/a.txt')`, _w1_ = `link('/', 'a.txt')`, _w2_ = `put('/a.txt')`
- _O2_: `update('/b.txt')`, _w3_ = `link('/', 'b.txt')`, _w4_ = `put('/b.txt')`
- _O3_: `remove('/c.txt')`, _w5_ = `rm('/c.txt')`, _w6_ = `unlink('/', 'c.txt')`

Suppose that we plan those operations out and it produces the following graph:

    Shard │
          │                ┌────────────┐
        A │                │ w4      w5 │
          │                └/──────────\┘
          │                /            \
          │   ┌───────────/┐            ┌\───┐
        B │   │ w1      w3 │            │ w6 │
          │   └───\────────┘            └────┘
          │        \
          │        ┌\───┐
        C │        │ w2 │
          │        └────┘

Some operations, like `update()`, can plan their item-level writes without
reading the shards first -- it always plans the same set of `link()` and `put()`
calls regardless of the current database state. Others, like `remove()`, need to
look at the current state in order to decide which `unlink()` calls they want to
make.

But even for unconditional operations like `update()`, we saw earlier that all
the shards they want to update must be loaded into memory _before_ any writes
are initiated, so that we know the version IDs of the shards before any of them
were changed. That means that execution of the above plan must begin with a read
for each affected shard, and only once those are complete can we being executing
the writes according to their dependency order. If this plan all executes
without failures, we get the following trace of requests:

    Shard │
          │   ┌─────────────┐                   ┌─────────────┐
        A │   │ READ ░░░░░░░│                   │ WRITE ░░░░░░│
          │   └─────────────┘                   ├─────────────┘
          │                                     └- w4, w5
          │   ┌─────────────┐   ┌─────────────┐                 ┌─────────────┐
        B │   │ READ ░░░░░░░│   │ WRITE ░░░░░░│                 │ WRITE ░░░░░░│
          │   └─────────────┘   ├─────────────┘                 ├─────────────┘
          │                     └- w1, w3                       └- w6
          │   ┌─────────────┐                   ┌─────────────┐
        C │   │ READ ░░░░░░░│                   │ WRITE ░░░░░░│
          │   └─────────────┘                   ├─────────────┘
                                                └- w2

Now supposed that the request for the [_w4_, _w5_] fails with a conflict. We
don't know which of _w4_ or _w5_ are actually involved in the conflicted item,
we just know someone else updated this shard while we were using it. This is one
downside of trying to bundle more changes into the same request. We must assume
that _w4_ and _w5_ were unsuccessful, and that the operations they belong to,
_O2_ and _O3_, have halted with a conflict.

At this point, the first thing we must do is re-read shard _A_ to obtain its new
version ID. We can assume the version IDs we hold for the other shards are still
valid without re-reading them. Once shard _A_ is reloaded, we need to replan all
the affected operations. Each operation needs to re-run its setup procedure --
_O3_, being a `remove()` call, might plan a different set of item changes given
the updated state of shard _A_. Therefore we throw all the planned writes for
these operations -- _w3_, _w4_, _w5_, _w6_ -- away, and generate new ones by
re-running the `update()` or `remove()` setup routine for each operation. In
this case, _O2_ will produce identical changes because `update()` is
unconditional, and let's say _O3_ also plans the same changes since it's
changing a child of the root directory so cannot do any more than one `unlink()`
call.

- _O2_: `update('/b.txt')`, _w7_ = `link('/', 'b.txt')`, _w8_ = `put('/b.txt')`
- _O3_: `remove('/c.txt')`, _w9_ = `rm('/c.txt')`, _w10_ = `unlink('/', 'c.txt')`

The new set of planned writes creates a new graph:

    Shard │
          │        ┌────────────┐
        A │        │ w8      w9 │
          │        └/──────────\┘
          │        /            \
          │   ┌───/┐            ┌\────┐
        B │   │ w7 │            │ w10 │
          │   └────┘            └─────┘

Once the conflicted shard has been reloaded and the remaining operations have
been replanned, we can begin executing the plan again. The full execution of
this scenario looks like this:

    Shard │                                                       │
          │   ┌─────────┐               ┌─────────┐   ┌─────────┐ │             ┌─────────┐
        A │   │ READ ░░░│               │ WRITE ░░│   │ READ ░░░│ │             │ WRITE ░░│
          │   └─────────┘               ├─────────┘   └─────────┘ │             ├─────────┘
          │                             └─ w4, w5 (conflict)      │             └─ w8, w9
          │   ┌─────────┐   ┌─────────┐                           │ ┌─────────┐             ┌─────────┐
        B │   │ READ ░░░│   │ WRITE ░░│                           │ │ WRITE ░░│             │ WRITE ░░│
          │   └─────────┘   ├─────────┘                           │ ├─────────┘             ├─────────┘
          │                 └─ w1, w3                             │ └─ w7                   └─ w10
          │   ┌─────────┐               ┌─────────┐               │
        C │   │ READ ░░░│               │ WRITE ░░│               │
          │   └─────────┘               ├─────────┘               │
                                        └─ w2                     │
                                                                  └─ replan

Note that _w1_ does not need to be replanned, even though it was in a request
with an affected operation _w3_. Only operations that belong to the same
`update()`/`remove()` calls as those in the conflicted request need to be
replanned. So, the second write request for shard _B_ contains a single write
_w7_, the replacement for _w3_.

Also note that, if the write request for _w2_ has not been started at the time
the replan happens, then _w2_ should remain part of the planned graph. The
operation it belongs to, _O1_, has not been invalidated by a conflict and so we
can assume its planned writes are still valid. Therefore the complete process
for handling a conflict is:

- Re-read the affected shard(s) to get their new version ID.
- Let Δ be the set of high-level Query calls that have item-level writes
  included in the conflicted request.
- Let δ be the set of item-level writes belonging to the operations in Δ.
- Let ω be the set of item-level writes that are in parts of the graph that have
  not begun to execute yet.
- Rerun the setup procedure for the operations in Δ, generating a new set of
  item-level writes ε.
- Build a new plan graph for the combined set of operations ω + ε - δ.
- Continue executing the modified plan.

In general, the replanning phase may be asynchronous. Consider our three write
operations:

- `update()` always performs the same set of `link()` and `put()` calls, without
  needing to read the shards first. It only needs the shards loaded into memory
  to prevent the race conditions with concurrent `remove()` calls.

- `remove()` needs to read the shards in order to plan which items it needs to
  `unlink()`. However, the set of shards it needs to read is always predictable:
  it needs to load the shards containing the document itself and all its
  ancestor directories.

- `prune()` needs to read an unpredictable set of shards by performing a
  `find()` call to discover all the documents it needs to delete. This may
  include loading shards that aren't already in the task's cache, because the
  conflict was caused by an `update()` adding new documents to the directory.

Once the conflicted shard has been reloaded, `update()` and `remove()` should
not cause any new shard reads to happen, and any information they need can be
read quickly from data in the task's shard cache. However, resetting a `prune()`
operation may incur additional I/O to read new shards. We would rather not pause
execution of all requests while we're waiting for this I/O and so the above
rules need a small amendment:

- When the conflict is first detected, immediately remove the operations δ from
  the pending graph so that it only contains operations for unconflicted Query
  calls. It may be simpler to build a fresh graph out of the operations in ω - δ
  than to actually remove δ operations from the existing graph.

- Once a conflicted operation's setup phase is complete and it generates a new
  set of item-level writes ε, we can incrementally add ε to the pending graph
  using the normal rules for adding operations. The pending graph may be
  different from what it was when the conflict first occurred, if some planned
  requests have begun executing since the conflict.

When adding new operations to a graph that's begun executing, we must only add
operations to groups that have not started executing yet. Once a request is
started for a write group, it should be locked and no further operations can be
added to it.
