# Purpose

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
