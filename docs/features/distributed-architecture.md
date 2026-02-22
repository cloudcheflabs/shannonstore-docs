# Distributed Architecture

ShannonStore follows a disaggregated architecture with clear separation between control plane and data plane:

## API Nodes (Control Plane)

- Handle all S3 protocol requests via the NIO HTTP server.
- Manage object metadata in partitioned RocksDB stores.
- Coordinate cluster operations (leader election, partition assignment, backup/restore).
- Run the Admin UI and Admin API on a separate Netty-based HTTP server.
- Perform erasure coding, encryption, and compression before distributing data to Data Nodes.

## Data Nodes (Data Plane)

- Store raw data chunks (erasure-coded shards) on local disks.
- Serve chunk read/write requests from API Nodes via a custom binary NIO RPC protocol.
- Report disk health and capacity to ZooKeeper for cluster-aware placement decisions.
- Support multi-disk configurations with hash-based chunk distribution.

## Communication

- API-to-Data Node communication uses a custom binary NIO protocol (Message) with a 19-byte header containing version, type, flags, and request ID fields. Message types include PUT_CHUNK,
  GET_CHUNK, COMMIT, ROLLBACK, DELETE_CHUNK, and various metadata/IAM/KMS sync operations.
- API Nodes maintain persistent connection pools (ConnectionPool) to Data Nodes for efficient RPC, with configurable pool size and acquire timeouts.

## Two-Port Architecture

- S3 Port (default 8080): Raw NIO HTTP server optimized for high-throughput object storage operations.
- Admin Port (default 8888): Netty-based HTTP server for the Admin UI, REST API, and cluster management endpoints. This separation ensures administrative operations don't compete with data
  path resources.