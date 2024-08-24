# Architecture

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
