# Read operations

VaultDB supports three main read operations:

- `get()`: read the value of a single document
- `list()`: read the names of a directory's children
- `find()`: recursively find the paths that sit inside a given directory

These may be combined into larger workflows, especially for performing a bulk
export of the database. This requires enumerating all the document paths in the
database, and looking up the value of each one.

Here we examine the read operations in more detail to determine the optimal way
to translate each of them into reads of the underlying shard files.
