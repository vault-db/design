# Storage model

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
