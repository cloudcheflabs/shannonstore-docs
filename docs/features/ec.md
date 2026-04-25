# Erasure Coding

ShannonStore uses Reed-Solomon erasure coding to protect data against hardware failures with minimal storage overhead.

- Objects are split into data shards and parity shards. Even if some shards are lost, the original data can be fully recovered.
- Default configuration tolerates up to 2 simultaneous shard failures with only 50% storage overhead, compared to 200–300% for traditional replication.
- Shards are automatically spread across different nodes and disks to maximize fault tolerance.
- When failures occur, only the missing shards are regenerated — no need to reassemble the entire object.
- Each shard carries a CRC32C checksum (see [Data Integrity](data-integrity.md)). Reconstruction never feeds a silently-corrupted shard into the decoder — mismatched shards are dropped and rebuilt from parity instead.
