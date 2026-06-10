# Bucket Counters (Object Count & Size)

Every bucket exposes two running counters — object count and total bytes — that update on every PUT, multipart-complete, and DELETE rather than being recomputed from a full metadata scan. The admin UI **Bucket Browser** uses them, and they are served alongside the bucket listing.

## How it works

Each API node maintains its own in-memory `ConcurrentHashMap<bucket, count>` and `ConcurrentHashMap<bucket, sizeBytes>` inside `BucketManager`. The node updates its own counter whenever it serves a write — under nginx round-robin, half the PUTs land on node A and half on node B, so each node carries half the delta.

When a client reads stats, the leader fan-outs `TYPE_GET_BUCKET_STATS` to every API node, each returns its local pair, and the caller sums them — that sum is the **cluster total** with no RocksDB scan.

```text
   GET /admin/browser/buckets
      │
      ▼
   ┌──────────────────────────────────────────────┐
   │ for each API node: ask "your delta?" via RPC │
   │   sum(count_i) → cluster object count        │
   │   sum(size_i)  → cluster total bytes         │
   └──────────────────────────────────────────────┘
```

Overwrites are tracked correctly: the writing node compares the new object size to the previous metadata and applies `Δsize = new - prev, Δcount = 0`. Deletes apply `Δsize = -prev.size, Δcount = -1` on the node that served the DELETE.

## Wire

The counters are surfaced as part of the bucket listing response:

```bash
curl -sS -H "Authorization: Bearer $TOK" \
  http://api:8888/admin/browser/buckets | jq .
[
  { "name": "lakehouse", "objectCount": 12842, "totalSizeBytes": 5012398476 },
  { "name": "scratch",   "objectCount": 0,     "totalSizeBytes": 0 }
]
```

Per-bucket details (storage class, EC profile) are on `/admin/buckets/{bucket}/storage-class` — see [Storage Classes & Tiering](storage-classes-tiering.md).

## What the counters are NOT

- **Not** per-prefix. The numbers are bucket-wide; a deep lakehouse with `iceberg/db/tbl/data/` subtrees does not get prefix-level stats from this counter. Per-prefix accounting is a future RFC.
- **Not** snapshot-replicated. Each node's counter is local. A node restart resets that node's contribution; the cluster total then drops by the restarted node's previous delta until it re-observes traffic. This is acceptable for an operations gauge; it is not a billing source of truth.
- **Not** authoritative across versioned DELETEs that write delete markers (the prior versions remain). Counters track *visible* objects.

## Bootstrap

A freshly imported bucket (counter = 0) triggers the per-node fullscan **once** on first stats request, seeds the counter from that scan, and serves O(1) from then on. So the first stats call after a cluster start is slower than subsequent ones — but only once per bucket per node.
