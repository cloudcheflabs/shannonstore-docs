# Metadata Management

## RocksDB-Based Indexing

- Object metadata (bucket, key, size, content type, version, chunk locations, etc.) stored in RocksDB with tunable parameters: 64MB write buffer, 128MB block cache, 10-bit bloom filters,
  16KB block size.
- Application-level value compression (SNAPPY/GZIP/ZSTD) prevents double-compression with RocksDB's built-in compression.

## Metadata Partitioning

- Configurable number of metadata partitions (default: 64).
- Partition formula: Math.abs((bucket + "/" + key).hashCode()) % numPartitions.
- Each partition has a primary API node and configurable replicas (default: 2).
- Partition-to-node assignment managed by PartitionManager and rebalanced when nodes join or leave.

## Two-Tier Caching

- In-memory cache (latestCache): ConcurrentHashMap for latest object versions, providing O(1) lookups without RocksDB I/O.
- RocksDB: Persistent storage with LRU block cache for warm data.
- TTL-based cache cleanup (configurable, default: 3600 seconds).

## Replication

- Writes go to the primary API node for the partition, then replicate asynchronously to replicas.
- Async callbacks trigger replication for both writes and deletions.
- Read path: local cache → primary node → replica nodes (fallback chain).