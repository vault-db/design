# Query interface and implementation

We'll now consider the operations provided by the Query interface, and how they
should be implemented. We will examine the machinery that the Shard Manager must
provide in order for Query operations to execute correctly and efficiently.

The Shard Manager does not assume anything about how many shard files exist and
what their names are. It receives requests to access particular shards from the
frontend Query API, and its job is to optimise these requests into as small a
number of reads/writes as possible to minimise latency while maintaining
consistency.




