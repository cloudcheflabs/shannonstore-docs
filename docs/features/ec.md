# Erasure Coding (Reed-Solomon)

ShannonStore uses Reed-Solomon erasure coding to provide data durability without the storage overhead of full replication.

## Algorithm

- Implements Reed-Solomon coding over Galois Field GF(2^8) with primitive polynomial 0x11D (x⁸ + x⁴ + x³ + x² + 1).
- Uses a Cauchy encoding matrix where the top portion is an identity matrix (systematic encoding — data shards are stored unmodified) and the bottom portion consists of Cauchy matrix
elements computed as 1/(x ⊕ y) in GF(2^8).
- Matrix inversion for reconstruction uses Gaussian elimination in GF(2^8).

## Configuration

- Default: 4 data shards + 2 parity shards (6 total). This means any 2 shards can be lost and the original data is still recoverable.
- Configurable via ecDataShards and ecParityShards properties.
- Storage overhead: 50% (6/4 = 1.5×), compared to 200-300% for 2-3× replication.

## Shard Distribution (Disk-Based)

- The DiskPlacementStrategy assigns shards across the cluster using a multi-pass algorithm:
- Pass 1: Select different nodes for each shard (maximizes fault tolerance across node failures).
- Pass 2: If more shards than nodes, allow same node but different disks (protects against disk failures).
- Pass 3: Fallback to reuse disks if necessary (logged as warning).
- Within each pass, disks are sorted by available space (descending) to balance storage utilization.

## Reconstruction

- When shards are missing (e.g., due to disk failure), ErasureCoder.reconstructShards() rebuilds them in-place without reassembling the original data — only the missing shards are
regenerated, making repair operations efficient.

