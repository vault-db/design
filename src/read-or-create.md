# Read-or-create

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
