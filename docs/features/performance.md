# Performance Optimization

ShannonStore is designed for high throughput and low latency through several optimization strategies.

- **Part Buffering**: Multipart uploads use a dual-layer buffer (in-memory + disk overflow) with encryption, and parts are pre-assigned to erasure coding shards to avoid re-encoding on completion.
- **Quorum Writes**: PUT operations return success as soon as enough data shards acknowledge, while parity shards continue writing in the background.
- **Streaming Prefetch**: Large object downloads use async prefetching to eliminate blocking while the client reads data.
- **Connection Pooling**: Persistent connections to Data Nodes are pooled and reused for efficient RPC communication.
- **Thread Pool Isolation**: Separate thread pools for S3 requests, chunk operations, encoding, and background tasks prevent resource contention.
