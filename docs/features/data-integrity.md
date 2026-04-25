# Data Integrity

ShannonStore protects every stored byte against silent corruption (bitrot) with end-to-end checksums, and includes an optional background scrubber for proactive verification of cold data.

## Per-Shard CRC32C

- A CRC32C checksum is computed for each erasure-coded shard at write time and stored in the object's metadata, so it travels with the shard reference and is replicated alongside the metadata partition.
- On every read, the shard's bytes are re-checksummed before being passed to EC decode. A mismatch causes the shard to be treated as missing, and the original data is reconstructed from parity instead — the corrupt bytes never reach the client.
- Hardware-accelerated CRC32C keeps the cost negligible on the hot path; existing objects without recorded checksums fall through verification and remain readable until rewritten.

## Bitrot Scrubber (optional)

- A background service that periodically reads every shard owned by the local API node, recomputes its CRC32C, and verifies it against the stored value. Mismatched shards are rebuilt from EC parity onto a healthy disk through the disk repair pipeline.
- **Disabled by default.** Toggled at runtime from the Admin UI (Maintenance → Bitrot Scrubber); the on/off decision is persisted to a small state file so it survives API node restarts without redeployment.
- Configurable scan interval, per-node concurrency, and chunk-per-second rate limit keep scrubbing off the hot path of normal client traffic.
- Cycles can be triggered manually from the Admin UI for ad-hoc verification, and live counters (scanned / corrupt detected / repaired / failed) are visible while a scan is running.
