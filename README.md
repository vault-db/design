# VaultDB internal design

This design sets out the data model, interface and internal workings of
[VaultDB][1]. Our aim is to produce a design that implements the desired
interface on top of the target storage platforms in as efficient a way as
possible, and to examine the consistency guarantees that we are able and not
able to achieve.

[1]: https://github.com/vault-db/vault-db


## Purpose

VaultDB is designed to store small numbers (hundreds or thousands) of small
objects, typically JSON documents or base-64 blobs, with all data being
encrypted at rest. Its primary application is storage of credentials --
passwords, TOTP keys, configuration parameters, and so on. The design has been
driven by the needs of such applications first and foremost.

The database is designed to run both server-side and in the browser, with the
backing storage provided by some manner of blob store such as the local
filesystem, [localStorage][2], or a cloud storage service like Dropbox or
[remoteStorage][3]. It assumes the backing store does not allow byte-level
access to files, but instead reads and writes files as a single unit; the only
way to update a file is to completely overwrite it with a new blob. There will
also usually be no way to co-ordinate across files using transactions, so writes
to files are assumed to be independent.

[2]: https://developer.mozilla.org/en-US/docs/Web/API/Window/localStorage
[3]: https://remotestorage.io/

The backing store will potentially be accessed over the network with high
latency, and so VaultDB's primary concern is that operations make as efficient
use of the network as possible while making a best effort at consistency. What
we mean by _consistency_ will be explored through the rest of this design.

VaultDB is not a long-lived server process but is instead implemented as a
JavaScript library. There is no central server that mediates requests, and a
single backing store can be accessed arbitrarily by any number of VaultDB
instances at a time. We expect that instances of the VaultDB client will be
short-lived, being used by command-line programs and short-lived web pages. So,
any in-memory state used by the database, such as cached data, will survive for
a short period of time before being dropped. The only persistent state is that
stored in the backing store.

We also expect that individual stores will see relatively small amounts of
traffic and infrequent concurrent access by independent processes. The main
intended application is credential storage for a single user, application or
small team, so all use of the database results from direct user interaction and
it's unlikely a single user will execute multiple database processes
concurrently. So when thinking about consistency, our main concern is partial
failure caused by failed network requests or interrupted program execution,
rather than data races caused by concurrent access. That being said, we
recognise that processes using VaultDB may live for a long time, and have reads
and writes separated by long periods of time, raising the likelihood of write
conflicts.

Workloads are expected to be read-heavy, that is objects will be read more often
than they are updated. Thus a priority in this design is to reduce the cost of
reads as the volume of stored data grows.


## Data model

Documents in VaultDB are arranged in a filesystem-like hierarchy rather than a
flat namespace. This provides a means to group related documents together so
that they can be found efficiently by a common prefix of their path. This
enables features like tab-completion and removal of whole sets of related
documents with a single function call. Let's define a few terms:

- _Document_: a blob of data, of arbitrary type. For the environments VaultDB is
  intended to operate in, this blob will typically be a string, either JSON or a
  base-64 string. However, our design doesn't assume any particular data
  serialisation.

- _Directory_: an object that groups one or more documents or directories
  together. Its value is a sorted list of the names of its children.

- _Path_: a complete ID that points to a document, or a prefix of such an ID.
  All paths begin with a slash (`/`). Document paths must not end with a slash.
  Directory paths, which are prefixes of document paths, must end with a slash.
  For example, `/alice/notes.txt` is a document path, and `/` and `/alice/` are
  directory paths.

- _Name_: the final segment of a path; the relative path of an item from its
  parent directory. Names do not begin with slashes, but must end with slashes
  if they identify a directory. `notes.txt` and `alice/` are names.

- _Item_: a complete logical database entry consisting of a path and either a
  document or directory; a key-value pair.

For example, a set of documents stored in VaultDB might logically resemble this
tree of files:

    /
    ├─┬ alice/
    │ └── notes.txt
    ├─┬ bob/
    │ └─┬ pictures/
    │   ├── avatar.jpg
    │   └── header.png
    └─┬ carol/
      └── profile.json

This tree contains four documents, whose paths are `/alice/notes.txt`,
`/bob/pictures/avatar.jpg`, `/bob/pictures/header.png`, and
`/carol/profile.json`. The item with path `/` is a directory and its value is
the names of its direct children: `['alice/', 'bob/', 'carol/']`. In total this
tree consists of nine items:

    | path                       | type      | value
    | -------------------------- | --------- | ------------------------------
    | `/`                        | directory | `['alice/', 'bob/', 'carol/']`
    | `/alice/`                  | directory | `['notes.txt']`
    | `/alice/notes.txt`         | document  | `<blob>`
    | `/bob/`                    | directory | `['pictures/']`
    | `/bob/pictures/`           | directory | `['avatar.jpg', 'header.png']`
    | `/bob/pictures/avatar.jpg` | document  | `<blob>`
    | `/bob/pictures/header.png` | document  | `<blob>`
    | `/carol/`                  | directory | `['profile.json']`
    | `/carol/profile.json`      | document  | `<blob>`

Strictly speaking, directories are redundant and their existence and values are
derived from the documents that exist. However, we want to store them explicitly
so that listing the documents with a common prefix is a single read and does not
require scanning the entire database.

We can think of directories as indexes of their children, and they should be
kept consistent -- the names stored in a directory item should exactly match the
set of documents that have that directory as a prefix. However, it is possible
that this consistency is broken, if a write task partially fails. In any case,
in the absence of transactions, we cannot guarantee consistency across multiple
reads. It is possible that listing a directory returns a name that does not
exist when we try to read it, for example. Since directories are derived data,
it should be possible to perform a consistency check via a full database scan as
a periodic maintenance task.


## Storage model

VaultDB is designed to run on top of _blob stores_; interfaces that let one read
and write files as a unit rather than having byte-level or page-level access to
data. This lets us run VaultDB on top of many interfaces available in the
browser, such as localStorage or cloud storage services. The only way to update
a file in such systems is to completely overwrite it with a new blob. This means
that typical database implementation methods such as B+trees or append-only logs
aren't especially useful as their advantages depend on _not_ reading and writing
an entire file for every operation.

The simplest possible implementation would be to store all the items in a single
file. This would simplify some operations, the main one being that it becomes
unnecessary to store directory items. If we have a complete copy of the database
as an atomic object, we can just scan all the document paths in the database for
those beginning with a given prefix. However, it is not desirable to read the
whole data set over the network and into memory for every operation we want to
do. As the volume of stored data grows, the cost of reading any single item from
the database will grow with it.

At the other extreme, we could store each item in its own file. This is tricky
if we want the _names_ of documents to be encrypted, not just their contents.
Designing a naming scheme that hides the names of items present, their
hierarchical structure, how often each item is accessed, and that allows
deterministic lookup and directory listing is quite hard to achieve. It would be
preferable to have item names encrypted along with their contents, using a
non-deterministic encryption with a random key and/or IV. This model also makes
it expensive to retrieve all the items in the database for a bulk export, as we
need to perform one network request for each item stored.

Between those extremes, there is _sharding_: the database is split into a fixed
number of files, and each item is allocated to a shard based on a hash of its
path. To make the shard assignment of each item less predictable, we actually
compute the HMAC of a path with a random key, rather than using a deterministic
hash. This model makes directories items necessary; without them we would have
to load all the shards into memory to scan them for documents matching a prefix,
and this is basically the same as the single-file model but with more
round-trips.

Sharding has benefits for security, as it reduces the amount of data that must
be loaded into memory to serve a single read. It improves read performance for
the same reason, as the amount of data that must be loaded to read a single item
can be kept roughly constant by increasing the number of shards as the database
grows. It and also allows more concurrent writes compared to the single-file
model where all writes are forced to be sequential. It works better than a
single file when the data is stored in a sync service like Dropbox or
remoteStorage, where it reduces the likelihood of data loss to conflicting
updates. However, this model requires more effort than the others to make sure
we make optimal use of the network when accessing multiple items, some of which
might reside in the same shard.

Of course, if an application wants to prioritise consistency over all other
concerns, it can configure the database to use a single shard, forcing all data
access operations to be sequential.


### Data layout

The exact serialisation of shard files does not affect our designs below; we
only assume that the various operations will need to read or write a shard based
on one or more data items residing in it, and the design doesn't depend on the
internal data layout. However we will summarise it here for completeness, though
the details may change in future.

- All databases contain a special file called the _root key_ file, that contains
  a set of cryptographic keys that are generated randomly when the database is
  first initialised. These keys are used for hashing item paths to shard IDs,
  and for encrypting individual items. These keys are themselves encrypted using
  an external secret, typically a passphrase run through a key derivation
  function. It may also contain other configuration data such as the current
  number of shards and their filenames.

- A shard file consists of a list of items separated by newlines (`\n`). Shard
  files may have arbitrary names, and there may be an arbitrary number of them.
  It is up to the VaultDB frontend to determine how many shards exist, and to
  map item paths onto shard IDs as it sees fit.

- The first item is a JSON _metadata object_ that contains the file format
  version, i.e. `{"version":1}`. This item is stored in plaintext.

- The second item is the _shard index_, a JSON array containing a sorted list of
  paths of all the items in the shard. This item is encrypted.

- The remaining items are the document/directory values, in the same order as
  their paths in the shard index. Each item is individually encrypted.

Here, by _encrypted_ we mean that an item is encrypted using a random key, the
_item key_. The item key is itself encrypted using the root keys and stored
along with the item itself. This chained encryption means the database's
passphrase can be changed by re-encrypting the root keys, and individual items
can be similarly re-encrypted by changing their item keys, without requiring
re-encryption of all the items in the store.

Encryption is accomplished using some appropriate authenticated encryption
scheme such as AES-GCM or XSalsa20-Poly1305, or an encrypt-then-MAC construction
using HMAC and AES-CBC. Keys and IVs are produced using cryptographically secure
random generators.

For example, if the first few items in our above example were stored in the same
shard, the file would look like this. All lines but the first would be encrypted
as stored as base-64 strings.

    {"version":1}
    ["/","/alice/","/alice/notes.txt","/bob/","/bob/pictures/"]
    ["alice/","bob/","carol/"]
    ["notes.txt"]
    <contents of /alice/notes.txt>
    ["pictures/"]
    ["avatar.jpg","header.png"]

This structure encrypts the item names separately from their data. Looking up an
item's value requires decrypting two items: the shard index, and the target item
itself. Listing a directory similarly requires decrypting the shard index and
the directory item, and no other document items. This minimises the amount of
material that must be decrypted in memory to read the desired data.

In the VaultDB frontend, all documents are assumed to be JSON values, but the
lower level machinery makes no assumptions about the internal layout of shard
files, the encoding of documents, the existence of different item types, or the
fact that items are encrypted. The job of the internal middleware is just to
make sure that reads and writes of shard files are optimal and correct, and that
network failures are correctly handled, given sufficient information from the
frontend about which shards it wants to access, and any causal relationships
between data accesses.


## Architecture

VaultDB consists of three layers:

- The _Query API_ is the public interface that applications will use; we will go
  through the operations it provides later in this design.

- The _Adapter API_ is an abstract interface with implementations for various
  different storage systems: the filesystem, localStorage, Dropbox,
  remoteStorage, etc.

- The _Shard Manager_, which sits between these two layers and optimises access
  to shard files -- that is, calls to the Adapter API -- across operations
  performed by the Query interface.

A great deal of this design deals with the implementation of the Shard Manager
and how it translates between the needs and capabilities of the other layers.


### Adapter types

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


## Query interface and implementation

We'll now consider the operations provided by the Query interface, and how they
should be implemented. We will examine the machinery that the Shard Manager must
provide in order for Query operations to execute correctly and efficiently.

The Shard Manager does not assume anything about how many shard files exist and
what their names are. It receives requests to access particular shards from the
frontend Query API, and its job is to optimise these requests into as small a
number of reads/writes as possible to minimise latency while maintaining
consistency.


### Read operations

VaultDB supports three main read operations:

- `get()`: read the value of a single document
- `list()`: read the names of a directory's children
- `find()`: recursively find the paths that sit inside a given directory

These may be combined into larger workflows, especially for performing a bulk
export of the database. This requires enumerating all the document paths in the
database, and looking up the value of each one.

Here we examine the read operations in more detail to determine the optimal way
to translate each of them into reads of the underlying shard files.


#### `get (DocPath) -> Doc`

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


#### `list (DirPath) -> [Name]`

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


#### `find (DirPath) -> [Path]`

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


### Write operations

VaultDB supports three write operations:

- `update()`: update the value of a single document, creating it if absent
- `remove()`: remove a single document
- `prune()`: recursively remove all the documents inside a given directory

These may be combined into larger workflows, especially for performing a bulk
write into the database. This can be very inefficient if a shard is read and
written for every item that needs to be updated in it, and it can also lead to
incorrect behaviour if these operations are not correctly ordered.

Here we examine the write operations in more detail to determine the optimal way
to translate each of them into reads and writes of the underlying shard files.


#### `update (DocPath, (Doc) -> Doc)`

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


#### `remove (DirPath)`

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

    `update`                          | `remove`
    --------------------------------- | ---------------------------------
    `list('/')`                       |
    `link('/', 'doc')`                |
    `get('/doc')`                     |
    `put('/doc', newDoc)`             |
                                      | `get('/doc')`
                                      | `rm('/doc')`
                                      | `list('/')`
                                      | `unlink('/', 'doc')`

If `update` and `remove` are executed independently by different clients, then
the operations within each one remain in the same order but the two sets of
operations can become interleaved. For example, this order of operations is also
possible:

    `update`                          | `remove`
    --------------------------------- | ---------------------------------
    `list('/')`                       |
    `link('/', 'doc')`                |
                                      | `get('/doc')`
                                      | `rm('/doc')`
                                      | `list('/')`
                                      | `unlink('/', 'doc')`
    `get('/doc')`                     |
    `put('/doc', newDoc)`             |

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

    `update`                          | `remove`
    --------------------------------- | ---------------------------------
    `list('/')`                       |
    `get('/doc')`                     |
    `link('/', 'doc')`                |
    `put('/doc', newDoc)`             |
                                      | `get('/doc')`
                                      | `list('/')`
                                      | `rm('/doc')`
                                      | `unlink('/', 'doc')`

To violate safety, we need to find a way to perform a write that produces an
unsafe state, but does not cause a CAS conflict, and so executes successfully.
In our previous example, this happened when `put()` was executed after
`unlink()`:

    `update`                          | `remove`
    --------------------------------- | ---------------------------------
    `list('/')`                       |
    `get('/doc')`                     |
    `link('/', 'doc')`                |
                                      | `get('/doc')`
                                      | `list('/')`
                                      | `rm('/doc')`
                                      | `unlink('/', 'doc')`
    `put('/doc', newDoc)` ⚠️           |

In this order of operations, `put()` produces a conflict, indicated by ⚠️,
because an `rm()` occurs between the `get()` and `put()` in `update`,
invalidating the `get()`. To avoid this, the `get()` in `update` would have to
happen after the `rm()` operation, for example:

    `update`                          | `remove`
    --------------------------------- | ---------------------------------
    `list('/')`                       |
                                      | `get('/doc')`
                                      | `list('/')`
                                      | `rm('/doc')`
    `get('/doc')`                     |
    `link('/', 'doc')`                |
                                      | `unlink('/', 'doc')` ⚠️
    `put('/doc', newDoc)`             |

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

    `update`                          | `remove`
    --------------------------------- | ---------------------------------
    `list('/')`                       |
                                      | `get('/doc')`
                                      | `list('/')`
                                      | `rm('/doc')`
    `get('/doc')`                     |
                                      | `unlink('/', 'doc')`
    `link('/', 'doc')` ⚠️              |
    `put('/doc', newDoc)` ⛔️          |

This invalidates the `list()` in `update` and makes `link()` fail with a
conflict, so we don't actually execute `put()` at all (indicated by ⛔️). This is
equivalent to the `remove` fully completing and the `update` not happening at
all, which is safe. The other possible ordering is that `unlink()` happens last:

    `update`                          | `remove`
    --------------------------------- | ---------------------------------
    `list('/')`                       |
                                      | `get('/doc')`
                                      | `list('/')`
                                      | `rm('/doc')`
    `get('/doc')`                     |
    `link('/', 'doc')`                |
    `put('/doc', newDoc)`             |
                                      | `unlink('/', 'doc')` ⚠️

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

    `update`                          | `remove`
    --------------------------------- | ---------------------------------
    `list('/')`                       |
                                      | `get('/doc')`
                                      | `list('/')`
                                      | `rm('/doc')`
    `get('/doc')`                     |
    `link('/', 'doc')`                |
    `put('/doc', newDoc)`             |
                                      | `unlink('/', 'doc')` ⚠️
                                      | `list('/')`
                                      | `unlink('/', 'doc')`

The second `unlink()` succeeds, but now we've caused a safety violation by
unlinking `/doc` when it still exists. Instead we need to redo any reads
affected by the conflict to (i.e. the `link()` call) to get their new state and
version IDs, and then redo all the writes for the `remove` function. We don't
need to redo reads for shards we've not seen a conflict on, for example, we
don't need to redo the `get()` call here, we can continue to use the version ID
we got from the `rm()` call, as long as there's no evidence that version is now
stale.

    `update`                          | `remove`
    --------------------------------- | ---------------------------------
    `list('/')`                       |
                                      | `get('/doc')`
                                      | `list('/')`
                                      | `rm('/doc')`
    `get('/doc')`                     |
    `link('/', 'doc')`                |
    `put('/doc', newDoc)`             |
                                      | `unlink('/', 'doc')` ⚠️
                                      | `list('/')`
                                      | `rm('/doc')`
                                      | `unlink('/', 'doc')`

This leaves the database in a safe state because `/doc` is removed before being
unlinked. If any other client updates `/` or `/doc` while this `remove` is
retrying, then we will get further conflicts and we can start over once again.

Now, recall that we're trying to find executions that lead to unsafe states but
don't trigger CAS conflicts. We just did this by looking at traces where `get()`
in `update` is not invalidated because it occurs after `rm()`. The other way to
avoid a conflict on the `/doc` shard would be for `rm()` to happen after
`put()`, that is:

    `update`                          | `remove`
    --------------------------------- | ---------------------------------
                                      | `get('/doc')`
                                      | `list('/')`
    `list('/')`                       |
    `get('/doc')`                     |
    `link('/', 'doc')`                |
    `put('/doc', newDoc)`             |
                                      | `rm('/doc')` ⚠️
                                      | `unlink('/', 'doc')` ⛔️

In this execution, the `get()` in `remove` is invalidated by the `put()`,
causing `rm()` to fail with a conflict. To avoid this, the `get()` in `remove`
would have to happen after `put()` as well:

    `update`                          | `remove`
    --------------------------------- | ---------------------------------
    `list('/')`                       |
    `get('/doc')`                     |
    `link('/', 'doc')`                |
    `put('/doc', newDoc)`             |
                                      | `get('/doc')`
                                      | `list('/')`
                                      | `rm('/doc')`
                                      | `unlink('/', 'doc')`

But this is just the two functions executing sequentially, so no race condition
is possible. Each function executes to completion, keeping the database in a
safe state at all times by the way their internal operations are ordered.

Other attempts to construct a safety violation without a conflict end in the
same way. For example, in order for the `unlink()` call not to cause a conflict
in the `update` function, it must happen either before the `list()` call in
`update` (which would be a sequential, non-interleaved execution), or it must
happen after the `link()` call, as in:

    `update`                          | `remove`
    --------------------------------- | ---------------------------------
                                      | `get('/doc')`
                                      | `list('/')`
                                      | `rm('/doc')`
    `list('/')`                       |
    `get('/doc')`                     |
    `link('/', 'doc')`                |
                                      | `unlink('/', 'doc')` ⚠️
    `put('/doc', newDoc)`             |

Calling `unlink()` here would be unsafe, as it would unlink a doc that is then
created via `put()`. Thankfully, the `unlink()` fails with a conflict because
`link()` invalidates the `list()` in `remove`. To avoid this, the `list()` in
`remove` would have to happen after `link()`:

    `update`                          | `remove`
    --------------------------------- | ---------------------------------
                                      | `get('/doc')`
    `list('/')`                       |
    `get('/doc')`                     |
    `link('/', 'doc')`                |
                                      | `list('/')`
                                      | `rm('/doc')`
                                      | `unlink('/', 'doc')`
    `put('/doc', newDoc)` ⚠️           |

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

    path             | type      | value
    ---------------- | --------- | ------------------
    `/`              | directory | `['path/']`
    `/path/`         | directory | `['a.txt', 'to/']`
    `/path/a.txt`    | document  | `<blob>`
    `/path/to/`      | directory | `['b.txt']`
    `/path/to/b.txt` | document  | `<blob>`

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

    path          | type      | value
    ------------- | --------- | -----------
    `/`           | directory | `['path/']`
    `/path/`      | directory | `['a.txt']`
    `/path/a.txt` | document  | `<blob>`

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

    path             | type      | value
    ---------------- | --------- | --------------------
    `/`              | directory | `['path/']`
    `/path/`         | directory | `['a.txt', 'to/']`
    `/path/a.txt`    | document  | `<blob>`
    `/path/to/`      | directory | `['b.txt', 'c.txt']`
    `/path/to/b.txt` | document  | `<blob>`
    `/path/to/c.txt` | document  | `<blob>`

Is it possible to execute these `remove()` and `update()` calls concurrently in
a way that leads to an unsafe state, by incorrectly deleting the `/path/to/`
directory? If these are executed sequentially, this is the complete set of
operations involved, where we have labelled each operation with a letter to make
them easier to refer to:

        | `update`                         |     |`remove`
    --- | -------------------------------- | --- | --------------------------------
        |                                  | _A_ | `list('/')`
        |                                  | _B_ | `list('/path/')`
        |                                  | _C_ | `list('/path/to/')`
        |                                  | _D_ | `get('/path/to/b.txt')`
        |                                  |     | `----`
        |                                  | _E_ | `rm('/path/to/b.txt')`
        |                                  |     | `----`
        |                                  | _F_ | `unlink('/path/', 'to/')`
        |                                  | _G_ | `unlink('/path/to/', 'b.txt')`
    _H_ | `list('/')`                      |     |
    _J_ | `list('/path/')`                 |     |
    _K_ | `list('/path/to/')`              |     |
    _L_ | `get('/path/to/c.txt')`          |     |
        | `----`                           |     |
    _M_ | `link('/', 'path/')`             |     |
    _N_ | `link('/path/', 'to/')`          |     |
    _P_ | `link('/path/to/', 'c.txt')`     |     |
        | `----`                           |     |
    _Q_ | `put('/path/to/c.txt', doc)`     |     |

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

        | `update`                         |     |`remove`
    --- | -------------------------------- | --- | --------------------------------
    _H_ | `list('/')`                      |     |
    _J_ | `list('/path/')`                 |     |
    _K_ | `list('/path/to/')`              |     |
    _L_ | `get('/path/to/c.txt')`          |     |
        | `----`                           |     |
    _M_ | `link('/', 'path/')`             |     |
    _N_ | `link('/path/', 'to/')`          |     |
        |                                  | _A_ | `list('/')`
        |                                  | _B_ | `list('/path/')`
        |                                  | _C_ | `list('/path/to/')`
    _P_ | `link('/path/to/', 'c.txt')`     |     |
        | `----`                           |     |
    _Q_ | `put('/path/to/c.txt', doc)`     |     |
        |                                  | _D_ | `get('/path/to/b.txt')`
        |                                  |     | `----`
        |                                  | _E_ | `rm('/path/to/b.txt')`
        |                                  |     | `----`
        |                                  | _F_ | `unlink('/path/', 'to/')`
        |                                  | _G_ | `unlink('/path/to/', 'b.txt')` ⚠️

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

        | `update`                         |     |`remove`
    --- | -------------------------------- | --- | --------------------------------
    _H_ | `list('/')`                      |     |
    _J_ | `list('/path/')`                 |     |
    _K_ | `list('/path/to/')`              |     |
    _L_ | `get('/path/to/c.txt')`          |     |
        | `----`                           |     |
    _M_ | `link('/', 'path/')`             |     |
    _N_ | `link('/path/', 'to/')`          |     |
        |                                  | _A_ | `list('/')`
        |                                  | _B_ | `list('/path/')`
        |                                  | _C_ | `list('/path/to/')`
    _P_ | `link('/path/to/', 'c.txt')`     |     |
        | `----`                           |     |
    _Q_ | `put('/path/to/c.txt', doc)`     |     |
        |                                  | _D_ | `get('/path/to/b.txt')`
        |                                  |     | `----`
        |                                  | _E_ | `rm('/path/to/b.txt')`
        |                                  |     | `----`
        |                                  | _G_ | `unlink('/path/to/', 'b.txt')` ⚠️
        |                                  |     | `----`
        |                                  | _F_ | `unlink('/path/', 'to/')` ⛔️

This modification to the previous execution means _F_ will no longer execute;
the barrier means that _F_ will only run if _G_ completes successfully. At no
point in this execution do we violate the requirement that all documents are
fully linked to the root directory.

To prevent the conflict on _G_, we would need _P_ not to happen between _C_ and
_G_. Remember we only perform _F_ if _P_ happens after _C_, so the only
situation worth investigating is where _P_ happens after _G_:

        | `update`                         |     |`remove`
    --- | -------------------------------- | --- | --------------------------------
    _H_ | `list('/')`                      |     |
    _J_ | `list('/path/')`                 |     |
    _K_ | `list('/path/to/')`              |     |
    _L_ | `get('/path/to/c.txt')`          |     |
        | `----`                           |     |
    _M_ | `link('/', 'path/')`             |     |
    _N_ | `link('/path/', 'to/')`          |     |
        |                                  | _A_ | `list('/')`
        |                                  | _B_ | `list('/path/')`
        |                                  | _C_ | `list('/path/to/')`
        |                                  | _D_ | `get('/path/to/b.txt')`
        |                                  |     | `----`
        |                                  | _E_ | `rm('/path/to/b.txt')`
        |                                  |     | `----`
        |                                  | _G_ | `unlink('/path/to/', 'b.txt')`
    _P_ | `link('/path/to/', 'c.txt')` ⚠️   |     |
        | `----`                           |     |
    _Q_ | `put('/path/to/c.txt', doc)` ⛔️  |     |
        |                                  |     | `----`
        |                                  | _F_ | `unlink('/path/', 'to/')`

Now _G_ falls between _K_ and _P_ and causes _P_ to fail with a conflict, and
thus _Q_ does not happen at all. `remove` runs to completion, and `update` fails
to create the document that would have triggered a safety violation.

To avoid this conflict, _K_ would need to happen after _G_. We can move _K_ down
below _L_, but the barrier below that necessitates moving all the other `update`
operations after this after _G_ as well.

        | `update`                         |     |`remove`
    --- | -------------------------------- | --- | --------------------------------
    _H_ | `list('/')`                      |     |
    _J_ | `list('/path/')`                 |     |
    _L_ | `get('/path/to/c.txt')`          |     |
        |                                  | _A_ | `list('/')`
        |                                  | _B_ | `list('/path/')`
        |                                  | _C_ | `list('/path/to/')`
        |                                  | _D_ | `get('/path/to/b.txt')`
        |                                  |     | `----`
        |                                  | _E_ | `rm('/path/to/b.txt')`
        |                                  |     | `----`
        |                                  | _G_ | `unlink('/path/to/', 'b.txt')`
    _K_ | `list('/path/to/')`              |     |
        | `----`                           |     |
    _M_ | `link('/', 'path/')`             |     |
    _N_ | `link('/path/', 'to/')`          |     |
    _P_ | `link('/path/to/', 'c.txt')`     |     |
        | `----`                           |     |
    _Q_ | `put('/path/to/c.txt', doc)`     |     |
        |                                  |     | `----`
        |                                  | _F_ | `unlink('/path/', 'to/')` ⚠️

Call _N_ now falls between _B_ and _F_ and causes _F_ to fail with a conflict,
and so we don't perform the problematic `unlink()` call and _Q_ is safe. We
can't move _N_ up before _B_ without also moving _K_ before _G_, due to the
barriers, so let's instead try moving _N_ to happen after _F_.

        | `update`                         |     |`remove`
    --- | -------------------------------- | --- | --------------------------------
    _H_ | `list('/')`                      |     |
    _J_ | `list('/path/')`                 |     |
    _L_ | `get('/path/to/c.txt')`          |     |
        |                                  | _A_ | `list('/')`
        |                                  | _B_ | `list('/path/')`
        |                                  | _C_ | `list('/path/to/')`
        |                                  | _D_ | `get('/path/to/b.txt')`
        |                                  |     | `----`
        |                                  | _E_ | `rm('/path/to/b.txt')`
        |                                  |     | `----`
        |                                  | _G_ | `unlink('/path/to/', 'b.txt')`
    _K_ | `list('/path/to/')`              |     |
        | `----`                           |     |
    _M_ | `link('/', 'path/')`             |     |
    _P_ | `link('/path/to/', 'c.txt')`     |     |
        |                                  |     | `----`
        |                                  | _F_ | `unlink('/path/', 'to/')`
    _N_ | `link('/path/', 'to/')` ⚠️        |     |
        | `----`                           |     |
    _Q_ | `put('/path/to/c.txt', doc)` ⛔️  |     |

_F_ now happens between _J_ and _N_, causing _N_ to fail with a conflict and
therefore _Q_ does not happen at all and we once again have a safe execution.
Avoiding this conflict would either require moving _F_ to happen after _N_,
which is just the preceding example, or moving it before _J_. Even moving _J_ as
late as possible and keeping _K_ after _G_, this gives us:

        | `update`                         |     |`remove`
    --- | -------------------------------- | --- | --------------------------------
    _H_ | `list('/')`                      |     |
    _L_ | `get('/path/to/c.txt')`          |     |
        |                                  | _A_ | `list('/')`
        |                                  | _B_ | `list('/path/')`
        |                                  | _C_ | `list('/path/to/')`
        |                                  | _D_ | `get('/path/to/b.txt')`
        |                                  |     | `----`
        |                                  | _E_ | `rm('/path/to/b.txt')`
        |                                  |     | `----`
        |                                  | _G_ | `unlink('/path/to/', 'b.txt')`
    _K_ | `list('/path/to/')`              |     |
        |                                  |     | `----`
        |                                  | _F_ | `unlink('/path/', 'to/')`
    _J_ | `list('/path/')`                 |     |
        | `----`                           |     |
    _M_ | `link('/', 'path/')`             |     |
    _P_ | `link('/path/to/', 'c.txt')`     |     |
    _N_ | `link('/path/', 'to/')`          |     |
        | `----`                           |     |
    _Q_ | `put('/path/to/c.txt', doc)`     |     |

This is basically the same as sequential execution. All the writes in `remove`
succeed because `update` hasn't performed any writes yet. `update` then performs
all the required `link()` calls to make the final write _Q_ safe. By trying to
construct an execution in which we end in an unsafe state, and then trying to
avoid the resulting conflicts, we've ended up with a sequential execution.

Let's take an even more extreme example where the entire `update` operation
happens between _G_ and _F_, which themselves cannot be re-ordered. Is it
possible for the call to _G_ to succeed, and still have _F_ happen erroneously?

        | `update`                         |     |`remove`
    --- | -------------------------------- | --- | --------------------------------
        |                                  | _A_ | `list('/')`
        |                                  | _B_ | `list('/path/')`
        |                                  | _C_ | `list('/path/to/')`
        |                                  | _D_ | `get('/path/to/b.txt')`
        |                                  |     | `----`
        |                                  | _E_ | `rm('/path/to/b.txt')`
        |                                  |     | `----`
        |                                  | _G_ | `unlink('/path/to/', 'b.txt')`
    _H_ | `list('/')`                      |     |
    _J_ | `list('/path/')`                 |     |
    _K_ | `list('/path/to/')`              |     |
    _L_ | `get('/path/to/c.txt')`          |     |
        | `----`                           |     |
    _M_ | `link('/', 'path/')`             |     |
    _N_ | `link('/path/', 'to/')`          |     |
    _P_ | `link('/path/to/', 'c.txt')`     |     |
        | `----`                           |     |
    _Q_ | `put('/path/to/c.txt', doc)`     |     |
        |                                  |     | `----`
        |                                  | _F_ | `unlink('/path/', 'to/')` ⚠️

Here _F_ fails with a conflict because _N_ happens between _B_ and _F_. We can
move _N_ before _B_, but that leaves a conflict on _P_ because _G_ happens
first. If we move _P_ before _G_ then _G_ fails instead, and prevents _F_ from
happening.

        | `update`                         |     |`remove`
    --- | -------------------------------- | --- | --------------------------------
        |                                  | _A_ | `list('/')`
        |                                  | _C_ | `list('/path/to/')`
        |                                  | _D_ | `get('/path/to/b.txt')`
    _H_ | `list('/')`                      |     |
    _J_ | `list('/path/')`                 |     |
    _K_ | `list('/path/to/')`              |     |
    _L_ | `get('/path/to/c.txt')`          |     |
        | `----`                           |     |
    _N_ | `link('/path/', 'to/')`          |     |
        |                                  | _B_ | `list('/path/')`
        |                                  |     | `----`
        |                                  | _E_ | `rm('/path/to/b.txt')`
        |                                  |     | `----`
        |                                  | _G_ | `unlink('/path/to/', 'b.txt')`
    _M_ | `link('/', 'path/')`             |     |
    _P_ | `link('/path/to/', 'c.txt')` ⚠️   |     |
        | `----`                           |     |
    _Q_ | `put('/path/to/c.txt', doc)` ⛔️  |     |
        |                                  |     | `----`
        |                                  | _F_ | `unlink('/path/', 'to/')`

Or, we can move _N_ after _F_, but that causes a conflict on _N_ that prevents
_Q_ from happening.

        | `update`                         |     |`remove`
    --- | -------------------------------- | --- | --------------------------------
        |                                  | _A_ | `list('/')`
        |                                  | _B_ | `list('/path/')`
        |                                  | _C_ | `list('/path/to/')`
        |                                  | _D_ | `get('/path/to/b.txt')`
        |                                  |     | `----`
        |                                  | _E_ | `rm('/path/to/b.txt')`
        |                                  |     | `----`
        |                                  | _G_ | `unlink('/path/to/', 'b.txt')`
    _H_ | `list('/')`                      |     |
    _J_ | `list('/path/')`                 |     |
    _K_ | `list('/path/to/')`              |     |
    _L_ | `get('/path/to/c.txt')`          |     |
        | `----`                           |     |
    _M_ | `link('/', 'path/')`             |     |
    _P_ | `link('/path/to/', 'c.txt')`     |     |
        |                                  |     | `----`
        |                                  | _F_ | `unlink('/path/', 'to/')`
    _N_ | `link('/path/', 'to/')` ⚠️        |     |
        | `----`                           |     |
    _Q_ | `put('/path/to/c.txt', doc)` ⛔️  |     |

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


#### `prune (DirPath)`

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


### Read-or-create

In addition to the core operations, we need one further special function for
dealing with the root key file. The first time the database is used, we need to
generate some random keys, encrypt them with the root passphrase, and write them
to storage. If a root key is already set, we need to read and decrypt it, and we
must not overwrite it.

For Locking adapters, this can be accomplished in one of two ways:

1. Using the base Locking interface, by locking the filename for the root keys
   and attempting to read it; if this fails then some new keys are generated and
   written, and the lock is released.

2. A more optimal solution can be provided by opening the root key file directly
   using `O_CREAT | O_EXCL`; this will fail if the file already exists and it
   can be read instead.

In the average case where the root keys already exist, both techniques require a
single `open()` call and a read of the file, but version 1 requires an
additional call to remove the lock.

For CAS adapters, this can be accomplished by attempting a write with no
version, which will fail if the file already exists, at which point we can read
it. Or, we can read it, and then attempt a write with no version if that fails,
which requires one less network call in the average case. Both versions require
retrying if the second request fails. There is no more optimal implementation
than this, so it may be of marginal benefit requiring Locking adapters to
provide a custom implementation of this function.

Doing the read first means we may not need to generate keys at all, so the input
to this function should possibly be a function that can produce the new value if
required, and is not called otherwise.


## Operation scheduling

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


### Rules required for correctness

Let's start with an example that models the above `update()` call:

- Operation _w1_ targets shard _B_ and has no dependencies
- Operation _w2_ targets shard _A_ and has no dependencies
- Operation _w3_ targets shard _A_ and depends on _w1_ and _w2_.

We can imagine building up a graph of these operations while trying to group
operations on the same shard. We'll start by placing _w1_ in the graph in a new
group for shard _B_.

    Shard │
          │
        A │
          │
          │
          │   ┌────┐
        B │   │ w1 │
          │   └────┘

Next, we place _w2_, which targets shard _A_, so we create a new group for that
shard:

    Shard │
          │   ┌────┐
        A │   │ w2 │
          │   └────┘
          │
          │   ┌────┐
        B │   │ w1 │
          │   └────┘

Finally, we need to add _w3_, which depends on the other two operations. It must
therefore be executed either _after_ or _at the same time as_ these operations
to achieve safety. It targets shard _A_, so our question is: can we put it in
the existing group containing _w1_, or must we create a new group? In this case,
_w3_ depends directly on _w1_ and it is safe to place it in the same group,
meaning _w1_ and _w3_ will be applied in the same write request.

We add _w3_ to the graph in the group with _w1_ and make dependency links
between the operations:

    Shard │
          │   ┌────────────┐
        A │   │ w2 ---- w3 │
          │   └────────/───┘
          │           /
          │      ┌───/┐
        B │      │ w1 │
          │      └────┘

The finished graph has a dependency between the [_w2_, _w3_] group and the
[_w1_] group, forcing the writes to be executed in that order:

    Shard │
          │                         ┌─────────────────┐
        A │                         │ WRITE ░░░░░░░░░░│
          │                         ├─────────────────┘
          │                         └─ w2, w3
          │   ┌─────────────────┐
        B │   │ WRITE ░░░░░░░░░░│
          │   ├─────────────────┘
              └─ w1

Now imagine that we add a fourth operation _w4_ that targets _B_ and depends on
_w3_. Can we put it in the same group as _w1_? No, we cannot: this would mean
executing _w1_ and _w4_ in one write, and then _w2_ and _w3_ in another. _w3_
happens after _w4_, or possibly not at all if the second write fails, so we've
violated the dependency order of _w4_ on _w3_. Therefore _w4_ needs to be put in
its own group, separate from _w1_.

    Shard │
          │   ┌────────────┐
        A │   │ w2 ---- w3 │
          │   └────────/──\┘
          │           /    \
          │      ┌───/┐    ┌\───┐
        B │      │ w1 │    │ w4 │
          │      └────┘    └────┘

That gives us the following write sequence:

    Shard │
          │                       ┌─────────────────┐
        A │                       │ WRITE ░░░░░░░░░░│
          │                       ├─────────────────┘
          │                       └─ w2, w3
          │   ┌─────────────────┐                     ┌─────────────────┐
        B │   │ WRITE ░░░░░░░░░░│                     │ WRITE ░░░░░░░░░░│
          │   ├─────────────────┘                     ├─────────────────┘
              └─ w1                                   └─ w4

This small example has given us some useful constraints for building graphs:

- If no write groups exist for a shard, a new group must be created.

- If operations _P_ and _Q_ target the same shard, and _P_ depends directly on
  _Q_, then _P_ may be placed in the same group as _Q_, but must not be placed
  in any earlier group.

- If an operation _P_ depends _indirectly_ on another operation _Q_, that is it
  depends on _Q_ via an operation on another shard, then _P_ must be placed in a
  later group than _Q_.

- Otherwise, operations may be assigned to any group we choose without breaking
  causality, assuming we record dependencies to force the groups to execute in
  the correct order.


### Assignment to the earliest group

If there are multiple groups that we're allowed to add an operation to, which
one do we choose? In general, it seems wise to choose the earliest-scheduled
group, that is the leftmost one in the graph. To see why, consider the following
example. Imagine we've built up the following graph, where:

- The operation _w1_ targets shard _B_.
- _w2_ targets shard _A_ and depends on _w1_.
- _w3_ targets _B_ and has no dependencies.
- _w4_ targets _C_ and depends on _w3_.
- _w5_ targets _B_ and depends on _w4_.

This gives us the following graph according to the above rules:

    Shard │
          │                 ┌────┐
        A │          .------- w2 │
          │         /       └────┘
          │        /
          │   ┌───/────────┐    ┌────┐
        B │   │ w1      w3 │    │ w5 │
          │   └───────────\┘    └/───┘
          │                \    /
          │                ┌\──/┐
        C │                │ w4 │
          │                └────┘

_w1_ and _w3_ have been merged into a group because they do not depend on each
other, so it is safe to execute them together as long as the other writes start
_after_ the write to _B_ is complete. The indirect dependency of _w5_ on _w3_
means it cannot be merged with _w3_ and must start a new group, otherwise it
would be committed before _w4_.

Now say we have another operation targeting _B_, _w6_. It has no dependencies
and we decide to merge it into the group with _w5_.

    Shard │
          │                 ┌────┐
        A │          .------- w2 │
          │         /       └────┘
          │        /
          │   ┌───/────────┐    ┌────────────┐
        B │   │ w1      w3 │    │ w5      w6 │
          │   └───────────\┘    └/───────────┘
          │                \    /
          │                ┌\──/┐
        C │                │ w4 │
          │                └────┘

Next we add _w7_, targeting _A_ and depending on _w6_. We can merge this with
_w2_ safely, with the effect that this group is now scheduled after all the
others.

    Shard │
          │                          ┌────────────┐
        A │          .---------------- w2      w7 │
          │         /                └────────/───┘
          │        /                         /
          │   ┌───/────────┐    ┌───────────/┐
        B │   │ w1      w3 │    │ w5      w6 │
          │   └───────────\┘    └/───────────┘
          │                \    /
          │                ┌\──/┐
        C │                │ w4 │
          │                └────┘

Finally we add _w8_ which targets _B_ and depends on _w4_ and _w7_. Its indirect
dependencies on _w3_ and _w6_ mean it cannot be merged into any existing group
without violating causality, and must be put in a group by itself.

    Shard │
          │                          ┌────────────┐
        A │          .---------------- w2      w7 │
          │         /                └────────/──\┘
          │        /                         /    \
          │   ┌───/────────┐    ┌───────────/┐    ┌\───┐
        B │   │ w1      w3 │    │ w5      w6 │    │ w8 │
          │   └───────────\┘    └/───────────┘    └/───┘
          │                \    /                 /
          │                ┌\──/┐                /
        C │                │ w4 ----------------'
          │                └────┘

This gives us the following execution for this set of writes, which preserves
the required ordering so that we do not start writing an operation until after
all its dependencies are successfully committed.

    Shard │
          │                                             ┌───────────┐
        A │                                             │ WRITE ░░░░│
          │                                             ├───────────┘
          │                                             └─ w2, w7
          │   ┌───────────┐               ┌───────────┐               ┌───────────┐
        B │   │ WRITE ░░░░│               │ WRITE ░░░░│               │ WRITE ░░░░│
          │   ├───────────┘               ├───────────┘               ├───────────┘
          │   └─ w1, w3                   └─ w5, w6                   └─ w8
          │                 ┌───────────┐
        C │                 │ WRITE ░░░░│
          │                 ├───────────┘
                            └─ w4

However, this execution is not optimal; we perform a total of five writes, all
sequentially, and _w2_ is scheduled much later than necessary, considering it
only depends on _w3_ being committed. What if instead we put _w6_ in the first
group, with _w1_ and _w3_?

    Shard │
          │                 ┌────┐
        A │          .------- w2 │
          │         /       └────┘
          │        /
          │   ┌───/────────────────┐    ┌────┐
        B │   │ w1      w3      w6 │    │ w5 │
          │   └───────────\────────┘    └/───┘
          │                \            /
          │                ┌\───┐      /
        C │                │ w4 ------'
          │                └────┘

Now, _w7_ can again be merged with _w2_, but now we do not delay _w2_ until
after _w5_ is committed.

    Shard │
          │                 ┌───────────┐
        A │          .------- w2     w7 │
          │         /       └───────/───┘
          │        /               /
          │   ┌───/───────────────/┐    ┌────┐
        B │   │ w1      w3      w6 │    │ w5 │
          │   └───────────\────────┘    └/───┘
          │                \            /
          │                ┌\───┐      /
        C │                │ w4 ------'
          │                └────┘

When we plan _w8_, we can now put it in the group with _w5_, on which it does
not depend. We cannot place _w8_ any earlier because of its indirect
dependencies on _w3_ and _w6_.

    Shard │
          │                 ┌───────────┐
        A │          .------- w2     w7 ------.
          │         /       └───────/───┘      \
          │        /               /            \
          │   ┌───/───────────────/┐    ┌────────\───┐
        B │   │ w1      w3      w6 │    │ w5      w8 │
          │   └───────────\────────┘    └/───────/───┘
          │                \            /       /
          │                ┌\───┐      /       /
        C │                │ w4 ------'-------'
          │                └────┘

We now have four writes rather than five, and two of them can be performed
concurrently whereas our previous plan required all writes to happen
sequentially, so the depth has been reduced from five to three.

    Shard │
          │                     ┌───────────────┐
        A │                     │ WRITE ░░░░░░░░│
          │                     ├───────────────┘
          │                     └─ w2, w7
          │   ┌───────────────┐                   ┌───────────────┐
        B │   │ WRITE ░░░░░░░░│                   │ WRITE ░░░░░░░░│
          │   ├───────────────┘                   ├───────────────┘
          │   └─ w1, w3, w6                       └─ w5, w8
          │                     ┌───────────────┐
        C │                     │ WRITE ░░░░░░░░│
          │                     ├───────────────┘
                                └─ w4

This suggests that when trying to merge an operation into an existing group, the
earliest or "leftmost" eligible group in the graph should be chosen, to reduce
the likely total depth of the graph. For "leaf" operations with no dependencies,
this means picking the first group at all times, but non-leaf operations must be
placed in groups to the right of any operations they depend on. In general,
operations should be scheduled as early as possible, but no earlier.

There is another important property at work here. Look again at the graph state
before we add _w6_:

    Shard │
          │                 ┌────┐
        A │          .------- w2 │
          │         /       └────┘
          │        /
          │   ┌───/────────┐    ┌────┐
        B │   │ w1      w3 │    │ w5 │
          │   └───────────\┘    └/───┘
          │                \    /
          │                ┌\──/┐
        C │                │ w4 │
          │                └────┘

The [_w1_, _w3_] group is already depended upon by the [_w2_] group, and so
adding _w6_ and _w7_ to these groups respectively does not create any new
inter-group dependencies that would block admission of new operations into the
first _B_ group. It was already impossible for anything depending on the [_w2_]
group to enter the [_w1_, _w3_] group, and the addition of _w6_ and _w7_ has not
changed that.

    Shard │
          │                 ┌───────────┐
        A │          .------- w2     w7 │
          │         /       └───────/───┘
          │        /               /
          │   ┌───/───────────────/┐    ┌────┐
        B │   │ w1      w3      w6 │    │ w5 │
          │   └───────────\────────┘    └/───┘
          │                \            /
          │                ┌\───┐      /
        C │                │ w4 ------'
          │                └────┘

So it is tempting to try to plan execution by avoiding the creation of new
inter-group dependencies, if possible. However, if we're considering each
operation one at a time while building the graph, when _w6_ is added we do not
yet know what other operations might come to depend on it.

    Shard │
          │                 ┌────┐
        A │          .------- w2 │
          │         /       └────┘
          │        /
          │   ┌───/────────────────┐    ┌────┐
        B │   │ w1      w3      w6 │    │ w5 │
          │   └───────────\────────┘    └/───┘
          │                \            /
          │                ┌\───┐      /
        C │                │ w4 ------'
          │                └────┘

Doing this sort of analysis requires comparing whole dependency sets and their
possible intersections, and every possible combination of groups they could
form, rather than building the graph one operation at a time, and is likely to
be more computationally expensive and harder to define an algorithm for it. So
for now we will take the simpler option of preferring to place each operation in
the earliest possible group.


### Avoiding excessively deep graphs

In VaultDB, operations will tend to have short dependency chains, typically with
only two levels, for example a `put()` depending on a set of `link()` calls.
`remove()` and `prune()` graphs may be deeper but these will normally be
executed rarely relative to `update()`. Despite this, it is possible to form
highly sub-optimal plans by following the rules we have so far. For example, we
could form a chain like the following:

    Shard │
          │   ┌────┐
        A │   │ w1 │
          │   └───\┘
          │        \
          │        ┌\──────────┐
        B │        │ w2     w3 │
          │        └──────────\┘
          │                    \
          │                    ┌\──────────┐
        C │                    │ w4     w5 │
          │                    └──────────\┘
          │                                \
          │                                ┌\──────────┐
        D │                                │ w6     w7 │
          │                                └──────────\┘
          │                                            \
          │                                            ┌\───┐
        E │                                            │ w8 │
          │                                            └────┘

This is constructed by following the rules for _w2_ depending on _w1_, _w4_ on
_w3_ and so on, and placing each new operation in the earliest existing group
for its shard. To complete eight operations we have _N_ = 5 and _D_ = 5, the
graph is completely sequential. While reducing _N_ is desirable, it is also good
to run requests in parallel where possible.

One way we could achieve this is by deciding to place _w5_ in a new group
_before_ that containing _w4_, giving us the following plan with _N_ = 6 and _D_
= 3.

    Shard │
          │   ┌────┐
        A │   │ w1 │
          │   └───\┘
          │        \
          │        ┌\──────────┐
        B │        │ w2     w3 │
          │        └──────────\┘
          │                    \
          │   ┌────┐           ┌\───┐
        C │   │ w5 │           │ w4 │
          │   └───\┘           └────┘
          │        \
          │        ┌\──────────┐
        D │        │ w6     w7 │
          │        └──────────\┘
          │                    \
          │                    ┌\───┐
        E │                    │ w8 │
          │                    └────┘

We still have some batched operations, but the graph is flatter and likely to
take less time to execute. It also allows for opportunistic merging; if the
request for group [_w5_] fails and must be retried, and we retry it after the
request for [_w2_, _w3_] completes, we can then fold _w5_ into the request for
[_w4_] because all the dependencies for _w4_ have completed. This is very much
a niche optimisation though, as normally shards have multiple groups as a result
of their dependencies _preventing_ the groups from merging, so a group is only
executed if all its previous groups were successfully committed. This dynamic
merging is only possible for disconnected subsets of the graph.

How would we implement this? We could keep track if the depth position of each
group, that is the length of the dependency chain to the left of a given group.
[_w1_] has depth 0 as it has no dependencies, [_w2_, _w3_] is depth 1, and
[_w4_] has depth 2. When we come to consider _w5_, we may decide not to add it
to a group of depth 2, but instead start a new depth-0 group for the same shard.

Whether to allow leaf operations to join depth-1 groups is a heuristic choice.
If we only place leaf operations in depth-0 groups, then we'd get the following
plan up to _w4_:

    Shard │
          │       ┌────┐                    N = 4, D = 2
        A │       │ w1 │
          │       └───\┘
          │            \
          │   ┌────┐   ┌\───┐
        B │   │ w3 │   │ w2 │
          │   └───\┘   └────┘
          │        \
          │        ┌\───┐
        C │        │ w4 │
          │        └────┘

Whereas if we allow leaf operations in depth-1 groups, then we get some group
merging without the long chain of stacked requests.

    Shard │
          │   ┌────┐                        N = 3, D = 3
        A │   │ w1 │
          │   └───\┘
          │        \
          │        ┌\──────────┐
        B │        │ w2     w3 │
          │        └──────────\┘
          │                    \
          │                    ┌\───┐
        C │                    │ w4 │
          │                    └────┘

This is especially useful in the case of "crossed" dependencies where we want to
update two documents where each document's parent directory is in the same shard
as the other document. For example:

- Shard _A_ contains document item `/alice/note` and directory item `/bob/`
- Shard _B_ contains document item `/bob/note` and directory item `/alice/`

We cannot complete these updates via a single write to each shard without
violating causality, but we can accomplish it in three requests:

- _w1_ represents `link('/alice/', 'note')` and targets shard `B`
- _w2_ represents `put('/alice/note')` and targets shard `A`; it depends on _w1_
- _w3_ represents `link('/bob/', 'note')` and targets shard `A`
- _w4_ represents `put('/bob/note')` and targets shard `B`; it depends on _w3_

These operations produce this graph if we allow leaf operations in depth-1
groups:

    Shard │
          │        ┌───────────┐
        A │        │ w2     w3 │
          │        └/─────────\┘
          │        /           \
          │   ┌───/┐           ┌\───┐
        B │   │ w1 │           │ w4 │
          │   └────┘           └────┘

If we don't allow leaf operations in depth-1 groups, then we still end up with
an _N_ = 3 graph because even though we will place _w3_ in its own group before
_w2_, we will merge _w4_ into the group with _w1_.

    Shard │
          │   ┌────┐           ┌────┐
        A │   │ w3 │           │ w2 │
          │   └───\┘           └/───┘
          │        \           /
          │        ┌\─────────/┐
        B │        │ w4     w1 │
          │        └───────────┘

The leaf operation _w1_ has ended up in a depth-1 group because we merged _w4_
with it, and _w4_ depends on _w3_. So we may as well allow leaf operations to be
placed in depth-1 groups to begin with.

This works because _w1_ and _w4_ target the same shard; if they targeted
different shards then we'd get the stair-step pattern we saw above. The
particular way the operations are grouped and ordered is affected by whether we
allow leaf operations into depth-1 groups or not, and this will affect how
future operations merge into the graph. Again, we don't want to try every
possible combination but instead pick rules that produce a reasonable plan with
a single pass over the set of operations.

We may also want to place restrictions on merging non-leaf operations into
existing groups. For example, consider a variation of an earlier example where
we've built the following graph, and we're about to add _w8_ which targets _C_
and depends on _w7_.

    Shard │
          │   ┌────┐
        A │   │ w5 │
          │   └───\┘
          │        \
          │        ┌\──────────┐
        B │        │ w6     w7 │
          │        └──────────\┘
          │                    \
          │   ┌────┐           ┌\   ┐
        C │   │ w1 │             w8
          │   └───\┘           └    ┘
          │        \
          │        ┌\──────────┐
        D │        │ w2     w3 │
          │        └──────────\┘
          │                    \
          │                    ┌\───┐
        E │                    │ w4 │
          │                    └────┘

If we merge _w8_ into the [_w1_] group, that will have the effect of delaying
all the groups for shards _C_, _D_ and _E_, and increase the graph's total depth
from 3 to 5. We may decide to place _w8_ in its own group of depth 2 than to
merge it into a depth-0 group, thereby increasing the depth of a bunch of
existing groups.

What if the operations on shards _D_ and _E_ didn't exist?

    Shard │
          │   ┌────┐
        A │   │ w2 │
          │   └───\┘
          │        \
          │        ┌\──────────┐
        B │        │ w3     w4 │
          │        └──────────\┘
          │                    \
          │   ┌────┐           ┌\   ┐
        C │   │ w1 │             w5
          │   └────┘           └    ┘

In this case, we'd probably want to merge _w5_ into the [_w1_] group, as doing
that doesn't increase the graph's total depth and it reduces the number of
groups by one. It's hard to make this call while building the graph, because we
don't know what further operations might be built on top of _w1_. Perhaps this
is an operation we could apply when we've finished building the graph, to merge
adjacent groups as long as one does not depend on the other, and merging them
doesn't increase the graph's total depth.

For example, if _w5_ is the final operation and we end up with a graph like
this:

    Shard │
          │   ┌────┐
        A │   │ w2 │
          │   └───\┘
          │        \
          │        ┌\──────────┐
        B │        │ w3     w4 │
          │        └──────────\┘
          │                    \
          │   ┌────┐           ┌\───┐
        C │   │ w1 │           │ w5 │
          │   └────┘           └────┘

Then we can reduce _N_ without affecting _D_ by merging the [_w1_] and [_w5_]
groups:

    Shard │
          │   ┌────┐
        A │   │ w2 │
          │   └───\┘
          │        \
          │        ┌\──────────┐
        B │        │ w3     w4 │
          │        └──────────\┘
          │                    \
          │                    ┌\──────────┐
        C │                    │ w5     w1 │
          │                    └───────────┘

But suppose further operations are added after _w5_ that depend on _w1_ and
target another shard:

    Shard │
          │   ┌────┐
        A │   │ w2 │
          │   └───\┘
          │        \
          │        ┌\──────────┐
        B │        │ w3     w4 │
          │        └──────────\┘
          │                    \
          │   ┌────┐           ┌\───┐
        C │   │ w1 │           │ w5 │
          │   └───\┘           └────┘
          │        \
          │        ┌\───┐
        D │        │ w6 │
          │        └────┘

In this case, merging the [_w1_] and [_w5_] groups would reduce _N_ by 1, but
would also increase _D_ by 1:

    Shard │
          │   ┌────┐
        A │   │ w2 │
          │   └───\┘
          │        \
          │        ┌\──────────┐
        B │        │ w3     w4 │
          │        └──────────\┘
          │                    \
          │                    ┌\──────────┐
        C │                    │ w5     w1 │
          │                    └──────────\┘
          │                                \
          │                                ┌\───┐
        D │                                │ w6 │
          │                                └────┘

This suggests that it would be profitable to avoid merging groups whose depth
differs by 2 or more (for example adding _w5_ to the [_w1_] group) while adding
operations to the graph. Once all operations have been added, we could look for
opportunities to merge groups for the same shard, as long as this does not
increase the graph's total depth, or increases it by less than some threshold if
we believe that reducing _N_ is worth increasing _D_ somewhat.

We could also consider approaches based on building the graph only after we have
all the dependency information for all writes, rather than attempting to build
the graph incrementally. Any graph-building algorithm will necessarily involve
an incremental phase to slot operations into the graph one by one, but can we
apply any analysis to the operations before starting this process that leads to
a better result?

For example, we could decide to sort the operations by their depth: leaf
operations are depth 0, operations that depend only on leaves are depth 1, and
so on. Take our stair-step plan from above:

    Shard │
          │   ┌────┐
        A │   │ w1 │
          │   └───\┘
          │        \
          │        ┌\──────────┐
        B │        │ w2     w3 │
          │        └──────────\┘
          │                    \
          │                    ┌\──────────┐
        C │                    │ w4     w5 │
          │                    └──────────\┘
          │                                \
          │                                ┌\──────────┐
        D │                                │ w6     w7 │
          │                                └──────────\┘
          │                                            \
          │                                            ┌\───┐
        E │                                            │ w8 │
          │                                            └────┘

Imagine we instead try to build a graph by adding all the depth-0 operations to
it first. These have no dependencies and will all produce a single group for
their respective shard.

    Shard │
          │   ┌────┐
        A │   │ w1 │
          │   └────┘
          │
          │   ┌────┐
        B │   │ w3 │
          │   └────┘
          │
          │   ┌────┐
        C │   │ w5 │
          │   └────┘
          │
          │   ┌────┐
        D │   │ w7 │
          │   └────┘
          │
          │
        E │
          │

Then we could handle the depth-1 operations. _w2_ depends on _w1_ and targets
shard _B_, so we can merge it into the [_w3_] group, increasing that group's
depth from 0 to 1. _w4_ depends on _w3_ and targets _C_, forcing the creation of
a depth-2 group. We don't merge _w4_ into the [_w5_] group because that would
increase the group's depth by 2.

    Shard │
          │   ┌────┐
        A │   │ w1 │
          │   └───\┘
          │        \
          │        ┌\──────────┐
        B │        │ w2     w3 │
          │        └──────────\┘
          │                    \
          │   ┌────┐           ┌\───┐
        C │   │ w5 │           │ w4 │
          │   └────┘           └────┘
          │
          │   ┌────┐
        D │   │ w7 │
          │   └────┘
          │
          │
        E │
          │

We can see how following this pattern produces a similar result to what we saw
above. It doesn't seem like sorting by operation depth produced a radically
different approach; graph-building decisions are still guided by the relative
depth of write groups. Since dependency chains in VaultDB are typically short
and msot operations have a depth of either 0 or 1, sorting them by depth doesn't
meaningfully change the graph building process. It just adds delay to the start
of the process, since we can't start building the graph until we have
information for all operations and we've sorted them. It also prevents graphs
being dynamically updated or optimised during execution, which is necessarily
for dealing with failed requests.

So far the best candidates for constraints to add to our rule set are:

- Do not add a leaf operation to the leftmost existing group if that group's
  current depth is greater than 1. Instead, create a new depth-0 group for the
  target shard.

- Do not add any operation to an existing group if doing so would increase the
  group's depth by more than 1. Instead, start a new group for the operation.

- Once all operations are known, attempt to merge any groups for a shard that
  are not mutually dependent. Merge groups as long as it does not increase the
  total graph depth unacceptably.


### Prioritising the longest chains

Let's consider another example where we'd want to constrain merging based on
group depth. Imagine we have the following graph:

    Shard │
          │        ┌───────────┐
        A │        │ w2     w3 │
          │        └/─────────\┘
          │        /           \
          │   ┌───/┐           ┌\───┐
        B │   │ w1 │           │ w4 │
          │   └────┘           └───\┘
          │                         \
          │                         ┌\───┐
        C │                         │ w5 │
          │                         └────┘

Now say we have operation _w6_ targeting _C_, and it has no dependencies. We
decide not to merge it with the [_w5_] group, since that has depth 3, and we
instead start a new depth-0 group for it.

    Shard │
          │        ┌───────────┐
        A │        │ w2     w3 │
          │        └/─────────\┘
          │        /           \
          │   ┌───/┐           ┌\───┐
        B │   │ w1 │           │ w4 │
          │   └────┘           └───\┘
          │                         \
          │   ┌────┐                ┌\───┐
        C │   │ w6 │                │ w5 │
          │   └────┘                └────┘

Again, if no further operations are added to this graph, we'd consider merging
the [_w6_] and [_w5_] groups to reduce the number of groups without increasing
depth. But imagine instead that we now add _w7_, which targets _B_ and depends
on _w6_. Which group do we add _w7_ to?

_w7_ does not depend in any way on _w1_ or _w4_, so we can add it to either of
their groups. If we pick the earlier group, the [_w1_] one, then link that group
to [_w6_], we end up increasing the depth of the graph to 5.

    Shard │
          │                     ┌───────────┐
        A │                     │ w2     w3 │
          │                     └/─────────\┘
          │                     /           \
          │        ┌───────────/┐           ┌\───┐
        B │        │ w7      w1 │           │ w4 │
          │        └/───────────┘           └───\┘
          │        /                             \
          │   ┌───/┐                             ┌\───┐
        C │   │ w6 │                             │ w5 │
          │   └────┘                             └────┘

However, if we placed _w7_ in the [_w4_] group, then linking this group to
[_w6_] does not increase the graph's depth and it remains at 4.

    Shard │
          │        ┌───────────┐
        A │        │ w2     w3 │
          │        └/─────────\┘
          │        /           \
          │   ┌───/┐   ┌────────\───┐
        B │   │ w1 │   │ w7      w4 │
          │   └────┘   └/──────────\┘
          │            /            \
          │       ┌───/┐            ┌\───┐
        C │       │ w6 │            │ w5 │
          │       └────┘            └────┘

Therefore when merging a non-leaf operation, we should pick a group whose depth
is greater than that of any of the groups the operation depends on. Remember
that it's the depth of _groups_ that matters here for the shape of the graph,
not the depth of operations within their own local dependency relationships.
This graph produces the following partly parallelised write sequence, and has
_N_ = 5 and _D_ = 4.

    Shard │
          │                   ┌─────────────┐
        A │                   │ WRITE ░░░░░░│
          │                   ├─────────────┘
          │                   └─ w2, w3
          │   ┌─────────────┐                 ┌─────────────┐
        B │   │ WRITE ░░░░░░│                 │ WRITE ░░░░░░│
          │   ├─────────────┘                 ├─────────────┘
          │   └─ w1                           └─ w7, w4
          │   ┌─────────────┐                                 ┌─────────────┐
        C │   │ WRITE ░░░░░░│                                 │ WRITE ░░░░░░│
          │   ├─────────────┘                                 ├─────────────┘
              └─ w6                                           └─ w5

This is a shorter graph depth than if we'd decided to merge _w6_ with _w5_, thus
pushing _w7_ into its own group, or than if we'd merged _w7_ with _w1_ as
described above. Both these other graphs would be fully sequential.

The results we end up with are sensitive to the order in which operations are
added, and the merging decisions we make as a result. Imagine that in this
example we'd instead ended up merging _w1_ and _w4_, rather than _w2_ and _w3_,
which is perfectly allowed by their dependency relationships.

    Shard │
          │   ┌────┐           ┌────┐
        A │   │ w3 │           │ w2 │
          │   └───\┘           └/───┘
          │        \           /
          │        ┌\─────────/┐
        B │        │ w4     w1 │
          │        └───────────┘
          │
          │
        C │
          │

With the information up to _w4_, there's no reason to prefer this over the
previous plan, they're structurally equivalent. The difference only becomes
apparent when we add more operations.

We then add _w5_, creating a group of depth 2 that depends on the depth-1 [_w4_,
_w1_] group, and we add _w6_ in its own depth-0 group rather than adding it to
the depth-2 [_w5_] group.

    Shard │
          │   ┌────┐           ┌────┐
        A │   │ w3 │           │ w2 │
          │   └───\┘           └/───┘
          │        \           /
          │        ┌\─────────/┐
        B │        │ w4     w1 │
          │        └───\───────┘
          │             \
          │   ┌────┐    ┌\───┐
        C │   │ w6 │    │ w5 │
          │   └────┘    └────┘

Then we add _w7_, which we can merge into the [_w4_, _w1_] group, linking that
group to [_w6_] creating a graph with _N_ = 5 and _D_ = 3.

    Shard │
          │          ┌────┐          ┌────┐
        A │          │ w3 │          │ w2 │
          │          └───\┘          └/───┘
          │               \          /
          │        ┌───────\────────/┐
        B │        │ w7     w4    w1 │
          │        └/─────────\──────┘
          │        /           \
          │   ┌───/┐           ┌\───┐
        C │   │ w6 │           │ w5 │
          │   └────┘           └────┘

This graph has _N_ = 5 like the ones before it, but now performs more of the
writes in parallel. The difference isn't big enough to be worth trying every
possible grouping of operations, it's more important to have an algorithm that
can make some reasonable optimisations that runs in linear time relative to the
number of operations. Sometimes we'll miss a slightly more optimal solution, but
that's preferable to making the optimiser much more complicated or have
super-linear performance scaling.

    Shard │
          │   ┌───────────────┐                       ┌───────────────┐
        A │   │ WRITE ░░░░░░░░│                       │ WRITE ░░░░░░░░│
          │   ├───────────────┘                       ├───────────────┘
          │   └─ w3                                   └─ w2
          │                       ┌───────────────┐
        B │                       │ WRITE ░░░░░░░░│
          │                       ├───────────────┘
          │                       └─ w1, w4, w7
          │   ┌───────────────┐                       ┌───────────────┐
        C │   │ WRITE ░░░░░░░░│                       │ WRITE ░░░░░░░░│
          │   ├───────────────┘                       ├───────────────┘
              └─ w6                                   └─ w5

Is there a method we could use to convert the _D_ = 4 solution for this set of
operations into the _D_ = 3 graph? Consider the graph shape we ended up with for
_D_ = 4:

    Shard │
          │        ┌───────────┐
        A │        │ w2     w3 │
          │        └/─────────\┘
          │        /           \
          │   ┌───/┐   ┌────────\───┐
        B │   │ w1 │   │ w7      w4 │
          │   └────┘   └/──────────\┘
          │            /            \
          │       ┌───/┐            ┌\───┐
        C │       │ w6 │            │ w5 │
          │       └────┘            └────┘

We can observe that the longest dependency chain between single operations is
the one connecting _w3_, _w4_ and _w5_. Imagine that we construct a new graph
containing just those operations, and split out the other operations into their
own sub-graphs:

    Shard │                                     │
          │   ┌────┐                            │        ┌────┐
        A │   │ w3 │                            │        │ w2 │
          │   └───\┘                            │        └/───┘
          │        \                            │        /
          │        ┌\───┐                       │   ┌───/┐      ┌────┐
        B │        │ w4 │                       │   │ w1 │      │ w7 │
          │        └───\┘                       │   └────┘      └/───┘
          │             \                       │               /
          │             ┌\───┐                  │          ┌───/┐
        C │             │ w5 │                  │          │ w6 │
          │             └────┘                  │          └────┘

We can then add the remaining operations back into the graph on the left. We
start by merging _w1_ into the [_w4_] group, since _w1_ has no dependencies and
the [_w4_] group has depth 1. Then we create a new group for _w2_ which depends
on _w1_:

    Shard │                                     │
          │   ┌────┐           ┌────┐           │
        A │   │ w3 │           │ w2 │           │
          │   └───\┘           └/───┘           │
          │        \           /                │
          │        ┌\─────────/┐                │               ┌────┐
        B │        │ w4     w1 │                │               │ w7 │
          │        └───\───────┘                │               └/───┘
          │             \                       │               /
          │             ┌\───┐                  │          ┌───/┐
        C │             │ w5 │                  │          │ w6 │
          │             └────┘                  │          └────┘

This leaves _w6_ and _w7_ to merge into the main graph. _w6_ gets a new group,
rather than adding it to the [_w5_] group which has depth 2. _w7_ can be added
to the [_w4_, _w1_] group and linked to the [_w6_] group.

    Shard │
          │          ┌────┐          ┌────┐
        A │          │ w3 │          │ w2 │
          │          └───\┘          └/───┘
          │               \          /
          │        ┌───────\────────/┐
        B │        │ w7     w4    w1 │
          │        └/─────────\──────┘
          │        /           \
          │   ┌───/┐           ┌\───┐
        C │   │ w6 │           │ w5 │
          │   └────┘           └────┘

By extracting the longest dependency chain between individual operations and
building the graph by starting with that chain, we reduce the total depth of the
graph. Here's another example of a graph that could be optimised in this way:

    Shard │
          │   ┌────┐            ┌────┐
        A │   │ w1 │            │ w6 │
          │   └───\┘            └/──\┘
          │        \            /    \
          │        ┌\──────────/┐    ┌\───┐
        B │        │ w2      w5 │    │ w7 │
          │        └────────/───┘    └────┘
          │                /
          │           ┌───/┐
        C │           │ w4 │
          │           └/───┘
          │           /
          │      ┌───/┐
        D │      │ w3 │
          │      └────┘

The dominant chain in this graph is the set of operations [_w3_, _w4_, _w5_,
_w6_, _w7_]. If we rebuild the graph starting with these operations, and then
add _w1_ and _w2_ back in, we end up with the following graph where _w1_ and
_w2_ are in their own groups because the existing groups for their shards have
too high a depth.

    Shard │
          │   ┌────┐               ┌────┐
        A │   │ w1 │               │ w6 │
          │   └───\┘               └/──\┘
          │        \               /    \
          │        ┌\───┐     ┌───/┐    ┌\───┐
        B │        │ w2 │     │ w5 │    │ w7 │
          │        └────┘     └/───┘    └────┘
          │                   /
          │              ┌───/┐
        C │              │ w4 │
          │              └/───┘
          │              /
          │         ┌───/┐
        D │         │ w3 │
          │         └────┘

However we could then merge the [_w1_] and [_w6_] groups without increasing the
total depth of the graph; the [_w2_] and [_w7_] groups both have depth 4 after
this merge.

    Shard │
          │                  ┌──────────────┐
        A │                  │ w6        w1 │
          │                  └/──\─────────\┘
          │                  /    \         \
          │             ┌───/┐    ┌\───┐    ┌\───┐
        B │             │ w5 │    │ w7 │    │ w2 │
          │             └/───┘    └────┘    └────┘
          │             /
          │        ┌───/┐
        C │        │ w4 │
          │        └/───┘
          │        /
          │   ┌───/┐
        D │   │ w3 │
          │   └────┘

Then we can also merge the [_w7_] and [_w2_] groups, because these operations
are independent. We're left with this graph that's reduced _N_ by 1 compared to
the graph we started with.

    Shard │
          │                  ┌───────────┐
        A │                  │ w6     w1 │
          │                  └/──\──────\┘
          │                  /    \      \
          │             ┌───/┐    ┌\──────\───┐
        B │             │ w5 │    │ w7     w2 │
          │             └/───┘    └───────────┘
          │             /
          │        ┌───/┐
        C │        │ w4 │
          │        └/───┘
          │        /
          │   ┌───/┐
        D │   │ w3 │
          │   └────┘

This type of optimisation assumes the total graph depth will be dominated by one
or a few long operation chains; in VaultDB this won't be especially useful since
most writes involve chains of just two operations, but this could be useful for
applying this planning algorithm to other problems. It might be useful to make
this optimisation an optional task to perform once the initial graph is built,
and before attempting to merge groups.


### Merging the final groups

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


### Handling failed requests

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
