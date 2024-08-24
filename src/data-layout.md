# Data layout

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
