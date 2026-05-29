# Erasure Coding

ShannonStore stores every object as Reed-Solomon erasure-coded shards rather than full replicas. The same logical guarantee — survive any *m* simultaneous disk or node losses — costs roughly half the bytes that 3× replication would, and rebuilds rebuild only the missing shards rather than the whole object.

```text
   PUT object (N bytes)
      │
      ▼
   ┌──────────────────────────────────────────┐
   │ split into k equal data shards (N/k each)│
   │ derive m parity shards from Reed-Solomon │
   │   over GF(2^8) — Vandermonde matrix      │
   └──────────────────────────────────────────┘
      │   (k = data shards, m = parity shards)
      ▼
   distribute k+m shards across data nodes / disks,
   ideally on distinct failure domains
```

Default install is **k = 4, m = 2** — read-quorum survives 2 simultaneous losses with 50% storage overhead. Configurable per cluster via `shannonstore.api.s3.ec.data.shards` and `shannonstore.api.s3.ec.parity.shards`. Trade-offs:

| Config | Storage overhead | Tolerated losses | Practical use |
| --- | --- | --- | --- |
| 2 + 1 | 50% | 1 | minimum viable; CI / dev clusters |
| 4 + 2 (default) | 50% | 2 | production sweet spot — 6 nodes, two-disk failure tolerance |
| 8 + 4 | 50% | 4 | high-availability cold storage, ≥12 nodes |
| 10 + 4 | 40% | 4 | archive-grade, ≥14 nodes |

The matrix is square per design choice: as long as **m ≤ k**, the storage overhead stays at *m/k* and the failure tolerance is *m* shards. Picking *m* higher than *k* gains nothing in tolerance but costs proportionally more bytes.

## How a PUT becomes shards

1. **Buffer the object** (or a part, for multipart) into N bytes.
2. **Split into k blocks** of `ceil(N/k)` bytes each. The last block is zero-padded; the original length is recorded in the object metadata so the trailer is stripped on read.
3. **Compute m parity blocks** by multiplying the k data blocks with a Vandermonde matrix over GF(2⁸). Any combination of k shards out of the k+m total is sufficient to reconstruct the original.
4. **Encrypt each shard** with a per-object data-encryption key from KMS (see [Encryption & Key Management](kms.md)).
5. **Place shards** across data nodes via Highest Random Weight (HRW) routing — the same key/shard pair lands on the same nodes deterministically, so a reader knows where to fetch without consulting a metadata service.
6. **Record the shard layout** in the object metadata: per-part array of `{chunkId, shardIndex, nodeAddress, diskPath, checksum}` rows, persisted on the API node's RocksDB and replicated to peers through the metadata replication channel.

The encoding step uses Intel ISA-L or a vectorized Java fallback depending on what's available at process start — the path is identical, only throughput changes.

## How a GET reconstructs the object

1. **Read the metadata** for the requested key (and version, if specified) from the local RocksDB.
2. **Fan out shard reads** to the k data shards first. The API node connects to each shard's owning data node through a pooled NIO client and streams the bytes back.
3. **Verify each shard's CRC32C** against the checksum recorded in metadata. A mismatch is treated as if the shard were missing — see [Data Integrity](data-integrity.md).
4. **If any data shard is missing or corrupt**, read enough parity shards to bring the total to k healthy shards, run the inverse Reed-Solomon decoder, and emit the reconstructed data.
5. **Cache the decoded part** in a bounded LRU keyed by `(objectKey, partNumber)`. A second range request that hits the same part doesn't re-decode — important for Iceberg / Parquet workloads where the column-footer and a column chunk are typically two reads against the same part.

The cache cap is in bytes, not entries — `shannonstore.api.decoded.part.cache.bytes`. Sized to roughly one part × the number of concurrent range readers you expect.

## Shard placement and failure domains

HRW (Rendezvous hashing) places shards such that:

- Adding a data node moves only the fraction of shards that map to the new node — typical churn is `1 / (existing+1)` per shard slot.
- Removing a data node forces re-replication of the shards that were on it, drawn from parity on the remaining nodes. The repair scheduler picks which target node receives each rebuilt shard using the same HRW so future reads find the shard without metadata lookup.
- Failure-domain awareness is opt-in via the per-node `rack` / `zone` tag. When set, the placement algorithm rejects layouts where multiple shards of the same `(key, part)` land in the same domain, falling back to a degraded placement only when the cluster has insufficient diverse domains.

The placement is deterministic given `{cluster topology, key, part}` — so two clients hitting the same object converge on the same shard set without coordination.

## Repair

Two paths feed the repair pipeline:

| Trigger | Source |
| --- | --- |
| Reactive | A GET found a missing or CRC-mismatched shard; the request used parity to satisfy the read and emitted a repair task for the failed shard. |
| Proactive | The optional [Bitrot Scrubber](data-integrity.md) periodically verifies every shard's CRC32C and emits repair tasks for any mismatch. |

`DiskRepairService` consumes the task queue:

1. Find a healthy disk on a different node than the failed shard's owner (placement constraints).
2. Reconstruct the missing shard from k other shards (data or parity).
3. Write the reconstructed bytes to the new disk; update the metadata's shard layout entry to point at the new `{nodeAddress, diskPath, checksum}`.
4. Replicate the updated metadata to peer API nodes.

The service throttles itself by `shannonstore.api.disk.repair.rate.limit.bytes.per.sec` so a large repair doesn't consume the inbound write bandwidth of the cluster.

## Operational sizing

A practical rule:

- **k + m ≤ healthy_data_nodes × disks_per_node × 0.9** — every shard slot wants a distinct disk if at all possible; the 0.9 fudge factor reserves room for one disk to die without forcing degraded placement.
- **k + m ≤ failure_domain_count × disks_per_domain** when failure-domain awareness is in use.

A 12-node cluster with 4 disks per node (= 48 shard slots) comfortably runs k=8 m=4 with failure-domain awareness across 4 racks. Smaller clusters should stick with k=4 m=2; the storage-saving from k=8 m=4 is identical (50%), but the operational headroom is much tighter.

## See also

- [Data Integrity](data-integrity.md) — the CRC32C and scrubber that protect EC shards from silent corruption.
- [Encryption & Key Management](kms.md) — the per-shard encryption applied after splitting and before placement.
- [Distributed Architecture](distributed-architecture.md) — how data nodes communicate the shards.
- [Disk Repair Service](disk-repair.md) — the repair queue consumer that rebuilds lost shards.
- [Multi-Disk Storage](multi-disk-storage.md) — the local-disk layout each data node serves shards from.
