# Performance Optimization

Five techniques together carry ShannonStore from the textbook "encode → fan out → reply" S3 pipeline to a production storage engine that holds throughput at line rate under concurrent load. Each technique addresses a specific waiting point — buffering for client backpressure, quorum for tail latency, prefetching for read pipelining, pooling for connection overhead, and thread isolation for fairness — and each is independently tunable.

## Part buffering

Multipart uploads expose a fundamental tension: the client treats parts as independent (any order, any concurrency), the storage layer wants them grouped (so a single EC pass spans the whole object). Without buffering, every part triggers its own EC encode at write time and another at `CompleteMultipartUpload` — twice the CPU.

ShannonStore solves this with a **two-layer buffer**:

```text
   UploadPart bytes
        │
        ▼
   ┌─────────────────────────────────────────┐
   │  In-memory buffer (bounded)              │
   │   - direct ByteBuffer for zero-copy     │
   │   - encrypted at-rest with per-part DEK │
   └────────┬────────────────────────────────┘
            │ overflow when memory cap hit
            ▼
   ┌─────────────────────────────────────────┐
   │  RocksDB-backed spill                   │
   │   - same encryption                      │
   │   - flushed every flushIntervalMs       │
   └─────────────────────────────────────────┘
            │
            ▼  CompleteMultipartUpload
   ┌─────────────────────────────────────────┐
   │  Single EC pass across all parts        │
   │  → final shards placed across cluster   │
   └─────────────────────────────────────────┘
```

Knobs:

| Key | Default | Effect |
| --- | --- | --- |
| `shannonstore.api.part.buffer.enabled` | `true` | Master switch. |
| `shannonstore.api.part.buffer.memory.max.bytes` | (configured) | In-memory cap per node before spilling. |
| `shannonstore.api.part.buffer.rocksdb.path` | `./data/s3-metadata/part-buffer` | Spill location. |
| `shannonstore.api.part.buffer.flush.interval.ms` | (configured) | Background flush cadence for the spill. |
| `shannonstore.api.part.buffer.peer.fetch.timeout.seconds` | (configured) | Timeout when CompleteMultipart needs a part buffered on a peer. |

The buffer is **cluster-aware**: a part uploaded to API node A and a completion call landing on API node B fetches the buffered bytes from A over the internal NIO plane, so SDKs can multipart-upload across a load balancer without sticky sessions.

Parts are also **pre-assigned to EC shard slots** at buffer time. The slot assignment uses the same HRW that final-write placement would, so at completion the per-part shards are already in their final layout — no re-encoding, just promotion from "buffered" to "committed".

## Quorum writes

A PUT replies as soon as **k of k+m shards acknowledge**. The remaining parity shards continue writing in the background. The math: with k=4 m=2 the client sees acknowledgement at 4-of-6, durability is already achieved (the 4 data shards reconstruct the object even if every parity write fails), and the perceived write latency drops from `max(all 6 sites)` to `4th-fastest of 6 sites`.

```text
   PUT object
       │  fan out 6 shard writes in parallel
       ├──► node-1  ─┐
       ├──► node-2  ─┤  4 of 6 acknowledged → reply 200 OK to client
       ├──► node-3  ─┤  (background continues for parities 5 and 6)
       ├──► node-4  ─┘
       ├──► node-5  (parity — still in flight)
       └──► node-6  (parity — still in flight)
```

Two corner cases the implementation handles:

1. **Background parity failure**: if a background parity write fails (target disk full, peer dies), the shard is enqueued for the [Disk Repair Service](disk-repair.md) on a different healthy disk. The client never sees the failure — they got their 200 OK already.
2. **Insufficient acks**: if fewer than k shards acknowledge within the per-shard timeout, the PUT fails with `503 SlowDown` and the partial layout is reaped. No half-written object is visible.

Quorum writes are always on; there's no opt-out. Operators tuning for strict durability semantics (k+m of k+m acks before reply) would gain nothing — k of k+m already saturates the durability guarantee given the EC parameters; the extra waits are pure latency.

## Streaming prefetch

Large GET responses use a producer/consumer split: a prefetch worker fans out shard reads in parallel and feeds a circular buffer; the response-writing thread consumes from the buffer as the client reads. The client's read rate becomes the only backpressure signal — fast clients see line-rate, slow clients see naturally throttled fetch.

| Key | Default | Effect |
| --- | --- | --- |
| `shannonstore.api.prefetch.queue.size` | `8` | Number of in-flight shard reads queued ahead of the client. |
| `shannonstore.api.prefetch.initial.count` | `8` | Shards fetched eagerly at GET start. |
| `shannonstore.api.s3.circular.buffer.size` | `64 MiB` | Per-request ring buffer. |
| `shannonstore.api.s3.small.object.threshold` | `64 MiB` | Below this, GET takes a simpler in-memory path. |

The prefetch path is what makes Range requests work well for Iceberg/Parquet: a column-chunk read of a few MiB lands in one shard, the prefetcher has already pulled it before the consumer asks, and the read returns from memory. A naive sequential GET would have re-fetched every range from disk.

## Connection pooling

Inter-node communication on port 9090 is a high-fan-out workload — a single client GET against a k=4 m=2 object triggers up to 6 peer reads, and a cluster serving thousands of QPS produces tens of thousands of concurrent peer connections. Without pooling, each request would TCP-handshake against each peer; the handshake cost would dominate the request budget.

`ConnectionPool` retains idle NIO clients per `(host, port)` and reuses them. Cap is per-peer, not global:

| Key | Default | Effect |
| --- | --- | --- |
| `shannonstore.api.connection.pool.size.per.host` | `64` | Idle clients retained per peer. |
| `shannonstore.api.connection.pool.acquire.timeout.seconds` | `30` | Blocking acquire deadline. |
| `shannonstore.api.connection.pool.warmup.enabled` | `true` | Pre-open at startup. |

Failed sockets are evicted immediately so a transient peer crash doesn't poison the pool. See [Network & Internal Communication](network.md) for the full pool semantics.

## Thread-pool isolation

Three independent pools serve three different waiting patterns:

| Pool | Sized via | Workload pattern |
| --- | --- | --- |
| Netty workers | `netty.worker.threads.multiplier × cpus` | IO multiplexing for 8080/8888 |
| Fetch pool | `fetch.thread.pool.size` (default `max(16, 4 × cpus)`) | Per-shard NIO reads on the internal plane |
| Chunk pool | `chunk.thread.pool.size` (default `max(16, 4 × cpus)`) | EC encode/decode, KMS unwrap, compression |

The split exists because doubling the wrong pool wastes the resource. Fetch threads spend nearly all their time blocked on sockets; doubling them gives more concurrency at near-zero CPU cost. Chunk threads spend nearly all their time on CPU; doubling them past `availableProcessors()` produces no gain. An operator confused about which pool to scale can read the metrics:

| Metric | Saturation hint |
| --- | --- |
| Fetch-pool acquire timeouts appearing in the API-node logs | fetch pool too small (or pool size per host too small) |
| `system.cpu.usage` pegged at 100% | chunk pool can't process incoming work; PUT/GET latency is CPU-bound |
| Netty event-loop time near 100% | netty pool too small for connection count |

## Object-size paths

The single piece of advice every operator should internalize:

| Object size | GET path | PUT path |
| --- | --- | --- |
| ≤ `s3.small.object.threshold` (default 64 MiB) | buffered in memory, decoded once, written to socket in one shot | buffered in memory, EC-encoded, written |
| > threshold | streaming with prefetch, decoded part-by-part, written to socket as decoded | streaming with quorum write |

The threshold is configurable, but the trade-off rarely changes: small objects benefit from the simpler in-memory path (no prefetch overhead, no ring-buffer copies), large objects benefit from streaming (constant memory footprint, prefetch concurrency). 64 MiB is a sensible cutoff — it's smaller than Parquet typical row groups (so a single Iceberg footer read takes the small path) and larger than checkpoint files (so a Spark checkpoint takes the small path).

## Workload tuning recipes

| Workload | Suggested overrides |
| --- | --- |
| Iceberg / Parquet analytics (read-heavy, range-heavy) | `prefetch.queue.size = 16`, `connection.pool.size.per.host = 128`, leave object compression `NONE` |
| Spark / Flink checkpoints (small frequent writes) | `s3.small.object.threshold = 128 MiB`, raise `part.buffer.memory.max.bytes`, ZSTD object compression |
| Backup / archive (large sequential writes) | leave defaults; raise `disk.repair.rate.limit` so background repairs from any disk failure during ingest don't lag |
| Time-series ingest (high QPS, small parts) | `netty.worker.threads.multiplier = 8`, `fetch.thread.pool.size = 4 × cpus`, ZSTD object compression |
| Mixed media (image / video heavy, low write rate) | object compression `NONE`, raise `chunk.thread.pool.size` only if profile shows EC saturation |

These are starting points — read the metrics, then tune. The defaults are already a reasonable middle ground.

## What's deliberately not optimised

Two paths take simple-and-correct over fast-but-complex:

- **Multi-range HTTP requests** (`Range: bytes=0-99,200-299`) are not supported. Implementing them well requires multipart MIME responses; SDKs that need multiple ranges issue them as separate requests, and the prefetch path handles that cleanly.
- **Coalesced metadata writes**. Each IAM mutation immediately fans out a snapshot rather than batching multiple mutations into one push. Batching would lower CPU but increases the leader→follower convergence window; the convergence guarantee is more valuable than the savings.

## See also

- [Erasure Coding](ec.md) — the encoding step the chunk pool drives.
- [Network & Internal Communication](network.md) — the connection pool and threading model in more depth.
- [Encryption & Key Management](kms.md) — the KMS unwrap that runs on the chunk pool.
- [Disk Repair Service](disk-repair.md) — the recipient of background-write failures from quorum writes.
- [Configuration Management](configuration.md) — the full set of knobs mentioned here.
- [Monitoring & Metrics](monitoring.md) — the gauges that tell an operator which knob to tune.
