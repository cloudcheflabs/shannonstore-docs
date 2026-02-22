# Data Compression

ShannonStore supports multiple compression algorithms to reduce storage costs and network bandwidth.

## Supported Algorithms

- SNAPPY: Optimized for speed with moderate compression ratio. Default choice for binary data.
- GZIP: Higher compression ratio, suitable for text-heavy content.
- ZSTD: Best compression ratio with good speed (level 3 by default). Falls back to GZIP if the zstd-jni library is unavailable.
- NONE: No compression applied.

## MIME-Type Based Compression

- Compression is selectively applied based on content type. Configurable list of compressible MIME types (e.g., text/*, application/json, application/xml).
- Binary formats (images, video, already-compressed archives) skip compression automatically.

## Auto-Selection

- When configured, the system samples the first 1KB of data to determine compressibility: if >80% of bytes are printable ASCII, GZIP is selected; otherwise SNAPPY is used. Files under 1KB
  skip compression entirely.

## Network Compression

- Separate from storage compression, applied to inter-node chunk transfers.
- Threshold-based: only chunks above networkCompressionThreshold (default 1KB) are compressed.
- The FLAG_COMPRESSED flag in the message header signals compressed payloads, enabling transparent decompression on the receiving side.

## RocksDB Value Compression

- Optional application-level compression of metadata values stored in RocksDB, avoiding double-compression with RocksDB's built-in compression.

