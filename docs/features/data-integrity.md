# Data Integrity

ShannonStore protects every stored byte against silent corruption (bitrot) with end-to-end checksums, and includes an optional background scrubber for proactive verification of cold data.

## End-to-end checksums

Every shard is checksummed when written and verified on every read. If a shard fails verification, the original data is reconstructed from erasure-coded parity instead — the corrupt bytes never reach the client. Verification is hardware-accelerated and adds no measurable latency to the hot path.

## Bitrot Scrubber (optional)

A background scrubber periodically reads every shard the cluster holds, verifies its checksum, and rebuilds anything corrupted from parity. Disabled by default; toggle from the admin UI per node, or trigger an ad-hoc cycle. Live counters (scanned / corrupt / repaired / failed) are visible while a scan runs. Scan interval and rate limit are configurable so scrubbing never competes with normal client traffic.
