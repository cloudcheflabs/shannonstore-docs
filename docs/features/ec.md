# Erasure Coding

ShannonStore stores every object as Reed-Solomon erasure-coded shards rather than full replicas. The same logical guarantee — survive any *m* simultaneous disk or node losses — costs roughly half the bytes that 3× replication would, and rebuilds rebuild only the missing shards rather than the whole object.

```text
   PUT object (N bytes)
      │
      ▼
   ┌──────────────────────────────────────────┐
   │ split into k equal data shards (N/k each)│
   │ derive m parity shards from Reed-Solomon │
   │   over GF(2^8) — Cauchy matrix           │
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
3. **Compute m parity blocks** by multiplying the k data blocks with a systematic encoding matrix over GF(2⁸) — a Cauchy matrix in the bottom (parity) rows, chosen because any square submatrix of a Cauchy matrix is invertible. Any combination of k shards out of the k+m total is sufficient to reconstruct the original.
4. **Encrypt each shard** with a per-object data-encryption key from KMS (see [Encryption & Key Management](kms.md)).
5. **Place shards** across data nodes via Highest Random Weight (HRW) routing — the same key/shard pair lands on the same nodes deterministically, so a reader knows where to fetch without consulting a metadata service.
6. **Record the shard layout** in the object metadata: per-part array of `{chunkId, shardIndex, nodeAddress, diskPath, checksum}` rows, persisted on the API node's RocksDB and replicated to peers through the metadata replication channel.

The encoding step is a pure-Java `ReedSolomon`/`ErasureCoder` implementation (`shannonstore-core`) — there is no Intel ISA-L or native/JNI acceleration in source today; throughput comes entirely from the JVM's own JIT.

## How a GET reconstructs the object

1. **Read the metadata** for the requested key (and version, if specified) from the local RocksDB.
2. **Fan out shard reads** to the k data shards first. The API node connects to each shard's owning data node through a pooled NIO client and streams the bytes back.
3. **Verify each shard's CRC32C** against the checksum recorded in metadata. A mismatch is treated as if the shard were missing — see [Data Integrity](data-integrity.md).
4. **If any data shard is missing or corrupt**, read enough parity shards to bring the total to k healthy shards, run the inverse Reed-Solomon decoder, and emit the reconstructed data.
5. **Cache the decoded part** in a bounded LRU keyed by `(objectKey, partNumber)`. A second range request that hits the same part doesn't re-decode — important for Iceberg / Parquet workloads where the column-footer and a column chunk are typically two reads against the same part.

The cache cap is in bytes, not entries — `shannonstore.api.s3.decoded.part.cache.bytes` (default 33554432 / 32 MiB, per `ApiConfig.java`). Sized to roughly one part × the number of concurrent range readers you expect.

## Shard placement and failure domains

HRW (Rendezvous hashing) places shards such that:

- Adding a data node moves only the fraction of shards that map to the new node — typical churn is `1 / (existing+1)` per shard slot.
- Removing a data node forces re-replication of the shards that were on it, drawn from parity on the remaining nodes. The repair scheduler picks which target node receives each rebuilt shard using the same HRW so future reads find the shard without metadata lookup.
- Failure-domain awareness is a 6-pass cascade (zone → rack → host → node → disk → cyclic reuse) implemented in `DiskPlacementStrategy` — see [Topology-Aware Placement](placement-cascade.md) for the full mechanism. Host-distinct placement works **out of the box** (a node's `hostId` defaults to its registration address); rack/zone-distinct placement is opt-in via the `rackId`/`zoneId` labels (`-Dshannonstore.rack.id=...` / `-Dshannonstore.zone.id=...`). Each pass only accepts disks that don't violate its distinctness rule, falling through to the next, more permissive pass when the cluster lacks enough diverse domains to fill the shard count.

The placement is deterministic given `{cluster topology, key, part}` — so two clients hitting the same object converge on the same shard set without coordination.

## Repair

Three paths feed shard reconstruction, and they aren't quite the "GET discovers, worker fixes" split you might expect:

| Trigger | Source |
| --- | --- |
| Reactive (write-time) | A PUT or multipart-part flush commits successfully with all *k* data shards acknowledged, but one or more *parity* shards failed to write (e.g. a [NIC silent failure](nic-fail-safety.md) on a parity target). `StorageService` enqueues the missing shard indices on `DiskRepairService`'s in-memory `reactiveQueue`; a dedicated worker thread drains it and rebuilds within seconds via `DiskRepairService.repairShards()`. This is a write-path safety net, not something a GET triggers. |
| Proactive (scheduled scan) | `DiskRepairService`'s own periodic scheduler (`shannonstore.api.repair.interval.seconds`, default 300s, when `shannonstore.api.repair.enabled=true`) scans locally-indexed metadata for `ChunkInfo` entries referencing disks that have been unhealthy longer than the grace period (`shannonstore.api.repair.grace.period.seconds`, default 600s) and reconstructs them. |
| Proactive (bitrot) | The optional [Bitrot Scrubber](data-integrity.md) periodically verifies every shard's CRC32C and, on mismatch, delegates directly to `DiskRepairService.repairShards()` to rebuild that shard. A *missing* chunk (rather than a checksum mismatch) is explicitly left to the scheduled scan above — the scrubber's own comment states this is "not the scrubber's concern." |

`DiskRepairService` (both the reactive-queue worker and the scheduled scanner) then:

1. Finds a healthy disk on a different node than the failed shard's owner (placement constraints).
2. Reconstructs the missing shard from k other shards (data or parity) via the same `ErasureCoder`.
3. Writes the reconstructed bytes to the new disk; updates the metadata's shard layout entry to point at the new `{nodeAddress, diskPath, checksum}`.
4. Replicates the updated metadata to peer API nodes.

There is no byte-rate throttle (`shannonstore.api.disk.repair.rate.limit.bytes.per.sec` does not exist in source) — the only concurrency control is `shannonstore.api.repair.max.concurrent` (default 4), which caps how many shard-reconstruction operations run at once rather than limiting bytes/sec.

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
