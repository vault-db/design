# VaultDB internal design

This design sets out the data model, interface and internal workings of
[VaultDB][1]. Our aim is to produce a design that implements the desired
interface on top of the target storage platforms in as efficient a way as
possible, and to examine the consistency guarantees that we are able and not
able to achieve.

[1]: https://github.com/vault-db/vault-db
