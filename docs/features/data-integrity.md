# Data Integrity

ShannonStore protects every stored byte against silent corruption (bitrot) with end-to-end checksums, and includes an optional background scrubber for proactive verification of cold data.

## End-to-end checksums

Every shard is checksummed when written and verified on every read. If a shard fails verification, the original data is reconstructed from erasure-coded parity instead — the corrupt bytes never reach the client. Verification is hardware-accelerated and adds no measurable latency to the hot path.

Objects in [block-streaming format](ec.md#block-streaming-format-2) are also verified per block. Each encrypted block carries an AES-GCM authentication tag. An unencrypted block has a CRC32C recorded in metadata. A block that fails either check is rebuilt from parity pieces before any of it is sent.

## Durability of writes

A PUT is acknowledged only after its shards are on disk, not just in the page cache:

- **Data nodes** fsync every shard file before acknowledging it (`shannonstore.data.storage.fsync.enabled`, default `true`). Without it, a power loss on several data nodes at once could lose acknowledged objects.
- **The part buffer** on API nodes (small single PUTs and multipart parts) syncs its write-ahead log on every write (`shannonstore.api.part.buffer.rocksdb.sync`, default `true`).
- A shard write the data node refuses (disk full, an I/O error) fails the shard. Before 2026-11-07 such a refusal was counted as a success.

Turning either setting off trades this guarantee for write throughput. Do so only when the hardware protects the cache, for example a battery-backed RAID controller.

## Bitrot Scrubber (optional)

A background scrubber periodically reads every shard the cluster holds, verifies its checksum, and rebuilds anything corrupted from parity. Disabled by default; toggle from the admin UI per node, or trigger an ad-hoc cycle. Live counters (scanned / corrupt / repaired / failed) are visible while a scan runs. Scan interval and rate limit are configurable so scrubbing never competes with normal client traffic.
