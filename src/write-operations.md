# Write operations

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
