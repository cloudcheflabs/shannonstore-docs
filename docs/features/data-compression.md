# Data Compression

ShannonStore supports multiple compression algorithms to reduce storage costs and network bandwidth.

- **Supported algorithms**: SNAPPY (speed-optimized), GZIP (higher ratio for text), ZSTD (best overall ratio), or NONE.
- Compression is selectively applied based on MIME type — binary formats like images and video are automatically skipped.
- Auto-selection mode samples data content to choose the best algorithm automatically.
- Network compression is applied separately for inter-node transfers above a configurable threshold.
