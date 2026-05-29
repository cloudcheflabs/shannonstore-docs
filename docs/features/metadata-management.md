# Metadata Management

Object metadata — the rows that say "key `X` in bucket `Y` has version `V`, lives on data nodes `[n1, n2, n3, n4, n5, n6]` at disk paths `[p1, …, p6]` with checksums `[c1, …, c6]` and an EC layout of k=4 m=2" — is the addressing layer that turns the cluster from a JBOD of unidentifiable bytes into an S3 endpoint. ShannonStore stores this metadata in a RocksDB-backed index on every API node, replicated across the API tier via a configurable mode (`PULL` / `SYNC` / `ASYNC_PUSH`), so any node can serve any request and a single node failure never hides an object.

```text
   Client PUT
        │
        ▼
   ┌──────────────────────────────────────────────────────┐
   │  API node — writeMetadata(meta)                       │
   │    1. persist to local IndexManager                  │
   │    2. trigger replication per current mode           │
   │       ─ PULL:  enqueue change-log entry              │
   │       ─ SYNC:  block until all owners ack            │
   │       ─ ASYNC_PUSH: fire-and-forget to peers         │
   └──────────────────────────────────────────────────────┘
                       │
   ┌───────────────────┴──────────────────────────────────┐
   │  Owner set for (bucket, key)                          │
   │   = top-2 HRW(api_nodes, key)                         │
   │   (replication factor configurable; default 2)        │
   └────────────────────┬─────────────────────────────────┘
                        │
        ┌───────────────┴─────────────────┐
        ▼                                 ▼
   API node owner-1                  API node owner-2
   ShardedIndexStore on               ShardedIndexStore on
   one or more local disks            one or more local disks
   (RocksDB per shard)                (RocksDB per shard)
```

The index is **sharded** at the local-disk level so RocksDB compaction never piles up on a single store, and **replicated** at the cluster level so every owner can serve a read without consulting a leader.

## Local storage layout

Each API node runs a `ShardedIndexStore` that hashes `(bucket, key)` to one of N internal RocksDB shards. The shards can be **striped across multiple physical disks** via configuration — useful for very large deployments where a single SSD can't keep up with the read/write rate on the index.

| Concern | Mechanism |
| --- | --- |
| Random read | shard chosen by hash, single RocksDB `Get` |
| Range scan (ListObjects) | reverse index keyed by `bucket + prefix` for efficient prefix walks |
| Compaction | RocksDB level-based; sharded so concurrent compactions don't conflict |
| Encryption at rest | values KMS-encrypted before `db.put()`, decrypted after `db.get()` |
| Backup | shard files snapshot as one set, see [Backup & Restore](backup.md) |

Per-disk striping uses the same `shannonstore.api.index.rocksdb.path` form as the single-disk variant, but accepts a comma-separated list. The store opens one column family per disk and rotates writes across them.

A small but important detail: the in-memory part-buffer index is **not** in the same store. Multipart uploads in flight live in a separate RocksDB at `shannonstore.api.part.buffer.rocksdb.path`. This separation keeps the hot path on the main index uncluttered by transient state that only matters until `CompleteMultipartUpload` lands.

## HRW routing

`(bucket, key) → owner set` is decided by Highest Random Weight (Rendezvous) hashing:

```
owners = top-N (api_nodes sorted by hash(node_id, bucket, key) desc)
```

with N = replication factor (default 2). HRW has two properties that matter:

- **Stable under churn**. Adding a new API node moves only the fraction of keys whose top-N shifted — typically `2 / (existing+1)` keys reshuffle to include the new node. No "rebalance window" exists; ownership migration is a side-effect of the next write or read against each affected key.
- **Decentralised**. Any node can compute who owns any key given the same `api_nodes` membership view. No leader is consulted on the read path.

## Replication modes

Three modes are available, configurable via `shannonstore.api.metadata.replication.mode`:

### `PULL` (default)

Writes commit locally, then enqueue a change-log entry. Owners pull their share at their own pace through a periodic catch-up loop.

```text
   Write on node A → commit local → reply 200 OK
                                      │
                                      └─► (later) owners pull through
                                          change-log replay
```

- **Strengths**: lowest write latency, naturally fault-tolerant — a dead peer doesn't slow writers, the change-log waits.
- **Trade-off**: a reader landing on a not-yet-pulled owner gets a brief miss window that triggers a transparent fetch from the writer.

### `SYNC`

Write commits locally, then blocks until every owner acknowledges receipt before replying.

```text
   Write on node A → commit local → push to owners → wait for ack
                                                       │
                                                       └─► reply 200 OK
```

- **Strengths**: read-after-write across the owner set is immediate. Every owner has the latest state by the time the client sees success.
- **Trade-off**: write latency is `max(owner_ack_time)`, so a slow peer slows every write. Use when strict freshness is required for cross-region or cross-AZ readers.

### `ASYNC_PUSH`

Write commits locally and fires push messages to peers without waiting for acks.

```text
   Write on node A → commit local → fire-and-forget to owners
                                  │
                                  └─► reply 200 OK
```

- **Strengths**: similar latency to PULL with less control-flow overhead — peers see the update sooner than PULL on a quiet cluster.
- **Trade-off**: lost messages aren't retried automatically. Used when the IAM/bucket-state sync channel is reliable and pull catch-up is overkill.

The default `PULL` is the safe choice for nearly every workload. `SYNC` is appropriate for cross-AZ deployments where read-your-writes must hold across owners; `ASYNC_PUSH` is rarely the right answer outside of tightly observed environments.

## Read path

A read for `(bucket, key)`:

1. Compute the HRW owner set.
2. If the local node is in the owner set, look up the local RocksDB — return immediately on hit.
3. If the local node is *not* in the owner set, fetch the metadata over the internal NIO plane from any of the owners.
4. The remote-fetch path includes a small cache (LRU on `(bucket, key) → metadata`) so subsequent reads against the same key on a non-owner don't keep round-tripping.

The cache is invalidated on receipt of any replicated update touching the key. Operators can size the cache via configuration; default is sufficient for most read-heavy workloads.

## Membership changes

When an API node joins or leaves the cluster:

| Event | Effect on metadata |
| --- | --- |
| New API node joins | HRW shifts; the new node is now a top-N owner for some fraction of keys; pulls those keys' metadata from current owners through the change-log catch-up |
| Existing API node leaves | HRW shifts; surviving owners absorb the keys the leaving node held; new replicas chosen from the remaining nodes pull the metadata; a "Cassandra-style cleanup" sweep can be enabled to drop local copies of keys an owner no longer holds |
| Leader change | irrelevant to metadata reads/writes; the leader matters only for IAM and KMS state |

The cleanup sweep is **opt-in** because it costs read amplification during the sweep window — the safer default is to let local copies linger until they're naturally evicted. Operators with disk-space constraints turn the sweep on; clusters with plenty of disk leave it off.

## Change log

A bounded ring buffer of recent metadata changes lives on each API node — `MetadataChangelog`. The PULL replication mode reads this log on owners to catch up; `ASYNC_PUSH` writes synthesize log entries on receipt. The log is sized by entry count (configurable) and is purely a coordination mechanism — durability of the metadata itself is provided by the RocksDB write, not by the change log.

When an owner falls far enough behind that the change log on the writer has rolled past the owner's last-seen position, the owner falls back to a full snapshot fetch through `listAllObjects` on the writer. This is the only path that costs O(cluster-size) bytes for a single peer to catch up; it's rare in practice and is logged loudly when it happens.

## Snapshot replication for bucket/IAM state

The object index above is per-`(bucket, key)`. The bucket-level configuration (versioning, ACLs) and IAM state (users, groups, policies, access keys) are smaller and ride a different channel: the full `BucketManager` and `AuthManager` snapshot is exported as a single blob on every mutation, KMS-encrypted, and broadcast to peer API nodes. Followers swap in the new blob atomically.

The two channels are kept separate because object metadata is per-key and high-throughput; bucket/IAM state is small and low-throughput. Mixing them would have either over-replicated the small state or under-replicated the large state.

## Object metadata shape

Per-object record (simplified — the full schema is in `ObjectMetadata`):

```text
{
  bucket:        "lake",
  key:           "year=2026/month=05/data.parquet",
  versionId:     "v-…",                       // null when versioning disabled
  size:          158_493_124,
  contentType:   "application/parquet",
  etag:          "d41d8cd98f00b204e9800998ecf8427e-7",
  lastModified:  1779999999999,
  seq:           42193714,                     // monotonic per writer for read-after-write
  deleted:       false,
  dataShards:    4,
  parityShards:  2,
  parts: [
    {
      partNumber:   1,
      size:         …,
      originalSize: …,
      md5:          "…",                       // input to composite ETag
      chunks: [
        { chunkId, shardIndex, nodeAddress, diskPath, checksum: <CRC32C> },
        ...                                    // k + m rows per part
      ]
    },
    ...
  ]
}
```

Two subtleties:

- `seq` is a monotonic counter assigned at commit time. PULL-mode catch-up uses it as the high-water mark — peers replay change-log entries until their `seq` matches the writer's. Read paths can also use it as a freshness guard ("read at seq ≥ N" for read-your-writes).
- `checksum` on the chunk row is CRC32C of the persisted shard bytes (not the object plaintext). [Data Integrity](data-integrity.md) covers how it's verified on every read.

## Diagnostics

Prometheus exposes:

| Metric | Meaning |
| --- | --- |
| `shannonstore.index.rocksdb.size.bytes` | per-disk RocksDB footprint |
| `shannonstore.index.rocksdb.gets.total` / `puts.total` | hot-path counters |
| `shannonstore.metadata.replication.lag.ms` | difference between writer's `seq` and the slowest owner's `seq` |
| `shannonstore.metadata.changelog.rollover.total` | full-snapshot fall-back trigger count |

A growing `replication.lag.ms` in PULL mode is the most common symptom an operator should investigate first — usually a peer's RocksDB compaction is unhealthily slow.

## See also

- [Distributed Architecture](distributed-architecture.md) — the cluster bootstrap and leader-election context.
- [Erasure Coding](ec.md) — the shard layout the per-part chunks point at.
- [Encryption & Key Management](kms.md) — what protects the metadata blob at rest.
- [Data Integrity](data-integrity.md) — the CRC32C the chunk rows store.
- [Monitoring & Metrics](monitoring.md) — the gauges around the replication lag.
- [Backup & Restore](backup.md) — how the index is snapshotted out of the cluster.
