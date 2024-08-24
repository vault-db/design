# Adapter types

Broadly speaking, adapters fall into two categories according to the type of
concurrency control they provide:

- _Locking_: before updating a resource, and possibly before reading it, the
  resource must be pre-emptively locked. Only the caller holding the lock is
  allowed to update the resource.

  For example, many SQL databases fall into this category. This strategy can be
  implemented on the filesystem as follows. In order to update the file
  `hello.txt`:

  - Attempt to open `hello.txt.lock` for writing using the flags `O_CREAT` and
    `O_EXCL`. If this fails, either bail or wait a bit and try again.
  - Read from the existing `hello.txt` if desired and if it exists.
  - Write new content to `hello.txt.lock`.
  - Close `hello.txt.lock` and rename it to `hello.txt`.

  This strategy makes sure updates to files are _exclusive_ (one caller is
  allowed to update the file at a time) and _atomic_ (the update is
  instantaneous, nobody reading the file will see a partially updated state).

- _Compare and Swap (CAS)_: reading or writing a file returns a _version_, which
  is a number, a content hash, a modification time or similar. Writes are only
  accepted if they're accompanied by a version that matches the file's current
  version.

  For example, CouchDB, the Dropbox API, and remoteStorage implement this
  strategy. It's more suitable than locking for high-latency remote network
  interaction, and can be implemented on top of a locking system somewhat
  easily, by storing a content hash or monotonic version number alongside the
  file, while holding a lock on it when it's being updated.

We can characterise these strategies as interfaces and plan implementations of
the Shard Manager on top of them. The Locking interface could look like this:

    interface Locking {
      lock (ShardId) -> Guard
    }

    interface Guard {
      read () -> Data | null
      write (Data) -> void
      release () -> void
    }

The `lock()` method waits and returns a `Guard` only when no other guards are
held for the given shard ID. The `Guard` allows arbitrary reads and writes to
the shard until `release()` is called, at which point it becomes unusable. A
risk with this implementation is that a process halts while holding an on-disk
lock, which needs to be expired somehow to allow future processes to make
progress.

The CAS interface looks like this:

    interface CAS {
      read (ShardId) -> (Data, Version) | null
      write (ShardId, Data, Version | null) -> (bool, Version)
    }

Reading returns some data and a version, and writing takes some data and a
version and returns true or false to indicate the write was accepted or not,
with the new version on success. On failure, the file should be read again to
get the latest data and version and the update should be retried.

These interfaces are deliberately minimal to make it easy to write new storage
adapters, and so that our Shard Manager design assumes as little as possible
about the underlying implementation.

All operations in these interface can also fail by throwing an exception (or
returning a rejected promise) for the following reasons:

- Network failure, e.g. attempting to access a server while offline
- Authorization failure, e.g. an access token is invalid or has expired
- Any other reason

Network and Authorization failures need to be explicitly modelled so we can take
specific application-level action in each case.

We expect most adapter implementations to use the CAS interface as they'll be
accessed over the network. The filesystem allows a Locking implementation, but
any Locking implementation can have a CAS built atop it. localStorage has
synchronous access and so allows implementation of either strategy. With that in
mind it may be most fruitful to bias towards good behaviour for CAS systems even
if a more optimal implementation for Locking systems is possible.
