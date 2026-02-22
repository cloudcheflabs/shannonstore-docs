# Network & Internal Communication

## Custom Binary NIO RPC Protocol

- 19-byte message header: version (1), type (1), flags (2), requestId (8), bodyLength (4), reserved (3).
- Request-reply pattern with requestId correlation for concurrent operations.
- Flag-based features: FLAG_COMPRESSED (network compression), FLAG_COMMIT (immediate chunk commit), FLAG_DISK_AWARE (targeted disk placement).
- Message types for all cluster operations: chunk CRUD, metadata replication, IAM/KMS sync, backup/restore, metrics collection, maintenance mode, log retrieval.

## Network Compression

- SNAPPY compression applied to payloads above configurable threshold.
- Transparent: sender sets FLAG_COMPRESSED, receiver automatically decompresses.

## Socket Tuning

- Configurable TCP send/receive buffer sizes (default: 2MB each).
- TCP_NODELAY enabled for low-latency responses.
- Direct ByteBuffer option for reduced memory copying.
