# Bucket Counters (Object Count & Size)

Every bucket shows its object count and total bytes in the admin UI's **Object Store** page and in the
bucket listing of the admin REST API. The numbers are exact, cover every API node, and survive restarts.

## How it works

Metadata is placed on the API nodes by rendezvous hashing (HRW): each `(bucket, key)` has an ordered
list of owners, and the first one is its **primary**. Every live object has exactly one primary, so:

1. Each API node counts, per bucket, the current (not deleted) objects it is the primary for, and sums
   their sizes. One pass over its metadata index produces the numbers for every bucket at once.
2. The node that answers `GET /admin/browser/buckets` asks every other API node for its share
   (`TYPE_GET_BUCKET_STATS` over the internal NIO channel) and adds them up.

```text
   GET /admin/browser/buckets
      │
      ▼
   ┌──────────────────────────────────────────────────────────┐
   │ for each API node: "how many objects are you primary for?" │
   │   sum(count_i) → bucket object count                       │
   │   sum(size_i)  → bucket total bytes                        │
   └──────────────────────────────────────────────────────────┘
```

Because the numbers come from the metadata itself, a restarted node reports the same share it had
before, and a node that served no traffic still counts the objects it owns.

A node reuses its last scan for `shannonstore.api.bucket.stats.cache.ms` (default `5000`) so that a
listing of many buckets costs one scan, not one per bucket. A write served by a node clears that node's
cached scan, so its own writes show on the next listing.

## Wire

```bash
curl -sS -H "Authorization: Bearer $TOK" \
  http://api:8888/admin/browser/buckets | jq .
[
  { "name": "lakehouse", "objectCount": 12842, "totalSizeBytes": 5012398476, "versioning": false },
  { "name": "scratch",   "objectCount": 0,     "totalSizeBytes": 0,          "versioning": true }
]
```

A non-administrator sees only the buckets their IAM policies let them list (`s3:ListBucket`).

Per-bucket details (storage class, EC profile) are on `/admin/buckets/{bucket}/storage-class` — see
[Storage Classes & Tiering](storage-classes-tiering.md).

## What the counters are not

- **Not** per-prefix. The numbers are bucket-wide.
- **Not** a count of versions. On a versioned bucket they count the current versions; noncurrent
  versions and delete markers are not included.
- **Not** instantaneous across nodes. Another node's share can be up to
  `shannonstore.api.bucket.stats.cache.ms` old.

!!! note "Changed in 1.0.0"
    Earlier builds kept in-memory counters that each node updated for the writes it served. They reset
    to zero when a node restarted, so a full bucket could show `0 Objects` after a rolling restart. The
    counts are now computed from the metadata as described above.
