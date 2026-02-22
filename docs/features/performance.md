# Performance Optimization

## Part Buffer System

- Dual-layer buffering for multipart uploads: in-memory LRU cache (MemoryPartBuffer, configurable max memory) with RocksDB overflow (LocalPartBufferStore).
- Buffered parts are envelope-encrypted (encrypted data + encrypted DEK + IV) for security.
- Parts are pre-assigned to EC shards during upload, enabling efficient CompleteMultipartUpload (metadata finalization only, no re-encoding).

## Quorum-Based Writes

- S3 PUT operations wait for only dataShards count (e.g., 4 of 6) to acknowledge before returning success to the client.
- Parity shard writes continue in the background, reducing write latency while maintaining durability.

## Streaming Downloads with Prefetch

- Large objects are streamed via CircularBufferInputStream (async producer-consumer pattern).
- Configurable prefetch queue fetches the next N parts while the client reads the current part, eliminating head-of-line blocking.

## Connection Pooling

- Persistent NioClient connections to Data Nodes with configurable pool size (default: 10 per host) and acquire timeout (default: 10 seconds).
- Connection pool warmup option for better startup performance.

## Thread Pool Isolation

- Separate executor pools for: S3 request handling (default: 512 threads), chunk fetch operations, EC encoding/chunk uploads, and background maintenance tasks.

