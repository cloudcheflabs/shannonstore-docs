# Backup & Restore

ShannonStore provides comprehensive backup and restore capabilities for metadata, IAM state, and KMS keystore.

## Metadata Backup

- Each API Node exports its RocksDB index partitions as compressed binary data (ZipOutputStream).
- Backup chunks are distributed to Data Nodes with configurable replication factor.
- Chunk ID format: partition-backup-{apiNodeId}-{partitionId}-{timestamp} for traceability.
- Backup operations are asynchronous and tracked via PartitionAction logging.

## IAM State Backup

- All users, groups, policies, and access keys serialized to JSON, encrypted with the KMS default key.
- Pushed to Data Nodes alongside metadata backups.
- Automatically restored on leader startup and synchronized to followers.

## KMS Keystore Backup

- Encrypted keystore uploaded to Data Nodes after key rotation events.
- Followers restore from Data Nodes during cluster join.
- End-to-end encryption ensures key material is never stored in plaintext outside memory.

## Restore Workflow

1. Download partition backups from Data Nodes.
2. Decompress (ZipInputStream) and import into local RocksDB via IndexManager.
3. Rebuild in-memory caches (latestCache, versionHistory).
4. Replicate restored metadata to replica nodes.
5. Task status tracked and reported: restore → success or failed.

## Smart Rebalance (5-Phase Orchestration)

1. Enable maintenance mode (block S3 traffic).
2. Flush all part buffers across the cluster.
3. Backup metadata on all nodes.
4. Rebalance partition assignments (leader-only operation).
5. Restore metadata on affected nodes and disable maintenance mode.

