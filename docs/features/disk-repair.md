# Disk Repair Service

ShannonStore includes an automatic disk repair system that detects failed disks and reconstructs lost shards using erasure coding parity.

## Dead Disk Detection

- The repair service periodically scans all object metadata on partitions where the API node is PRIMARY.
- For each chunk, it checks whether the disk key (nodeAddress:diskPath) exists in the current set of healthy disks (reported by Data Nodes to ZooKeeper).
- A grace period (default: 300 seconds) prevents premature repairs for transient disk unavailability (e.g., temporary network issues or disk remounts).

## Reconstruction Process

1. Collect Surviving Shards: Fetch surviving shards from healthy Data Nodes in parallel via TYPE_GET_CHUNK messages, with configurable timeout (default: 60 seconds).
2. Verify Quorum: At least dataShards count of surviving shards must be available for reconstruction.
3. Reconstruct Missing Shards: Uses ErasureCoder.reconstructShards() which inverts the submatrix formed by present shards and applies matrix multiplication to regenerate only the missing
   shards — no need to reassemble the original data.
4. Write Repaired Shards: Select new healthy disks using DiskPlacementStrategy (preferring different nodes, then different disks). Send repaired shards via TYPE_PUT_CHUNK with
   FLAG_DISK_AWARE to target specific disks.
5. Update Metadata: Modify ChunkInfo with the new node address and disk path, persist to RocksDB, and replicate to replica API nodes.

## Operational Controls

- Configurable scan interval (default: 60 seconds between repair cycles).
- Configurable maximum concurrent repair threads (default: 8).
- Minimum available bytes threshold (default: 100MB) to prevent writing to nearly-full disks.
- Manual trigger available via Admin UI or API endpoint.
- Complete audit trail: every repair operation is logged as a PartitionAction with timestamps, affected counts, and status.

