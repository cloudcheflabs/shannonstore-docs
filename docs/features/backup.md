# Backup & Restore

ShannonStore provides comprehensive backup and restore for metadata, IAM state, and encryption keys.

- **Metadata Backup**: Each API Node exports its metadata partitions as compressed data, distributed to Data Nodes with configurable replication.
- **IAM State Backup**: All users, groups, policies, and access keys are serialized, encrypted, and stored on Data Nodes.
- **KMS Keystore Backup**: Encrypted keystores are backed up to Data Nodes after key rotation events.

## Restore

- Backups are downloaded from Data Nodes, imported into local stores, and in-memory caches are rebuilt.
- Restored metadata is replicated to replica nodes for consistency.

## Smart Rebalance

A 5-phase orchestrated workflow for safe partition rebalancing: enable maintenance mode → flush buffers → backup metadata → rebalance partitions → restore and resume operations.
