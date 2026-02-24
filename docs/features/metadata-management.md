# Metadata Management

ShannonStore manages object metadata through a partitioned, replicated, and cached architecture.

- Metadata is stored in RocksDB and distributed across configurable partitions, each with a primary node and replicas for fault tolerance.
- A two-tier caching layer (in-memory + RocksDB block cache) provides fast lookups without disk I/O for frequently accessed objects.
- Partition assignments are automatically rebalanced when nodes join or leave the cluster.
- Writes replicate asynchronously to replicas; reads follow a fallback chain from local cache to primary to replicas.
