# Data model

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
