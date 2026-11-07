# Performance Optimization

Five techniques together carry ShannonStore from the textbook "encode → fan out → reply" S3 pipeline to a production storage engine that holds throughput at line rate under concurrent load. Each technique addresses a specific waiting point — buffering for client backpressure, quorum for tail latency, prefetching for read pipelining, pooling for connection overhead, and thread isolation for fairness — and each is independently tunable.

## Part buffering

Multipart uploads expose a fundamental tension: the client treats parts as independent (any order, any concurrency), the storage layer wants them grouped (so a single EC pass spans the whole object). Without buffering, every part triggers its own EC encode at write time and another at `CompleteMultipartUpload` — twice the CPU.

ShannonStore solves this with a **write-back part buffer** on each API node:

```text
   UploadPart bytes (always)  /  single-PUT bodies up to part.buffer.max.bytes (256 KiB)
        │  encrypted with a per-part DEK (envelope encryption)
        ▼
   ┌────────────────────────────────────────────────┐
   │  Local RocksDB part buffer                      │
   │   - WAL on: the client is answered once here    │
   │   - large values in blob files (BlobDB)         │
   │   - small header record per part (no payload)   │
   └────────┬───────────────────────────────────────┘
            │ flusher, every flush.interval.ms,
            │ distinct objects in parallel (flush.parallelism)
            ▼
   ┌────────────────────────────────────────────────┐
   │  EC encode → shards to data nodes → metadata    │
   │  marked committed (only the header is rewritten)│
   └────────────────────────────────────────────────┘

   Larger single PUTs skip the buffer: EC-encoded and written to the data nodes
   before the client is answered.
```

**Only small single PUTs are buffered; multipart parts always are.** The buffer is what carries an in-flight multipart upload across an API-node restart, so every part goes through it. For single PUTs, the buffer answers small writes quickly, but every buffered byte is written again locally (WAL, blob/SST file) and flushed later, so for large payloads it costs more than writing the shards directly, and under sustained uploads the flush backlog grew without bound. Measured on the docker compose test cluster: buffering wins at 4 KiB (about 1,350 vs 990 PUT/s); from 1 MiB, writing directly is faster (349 vs 225 MiB/s at 1 MiB, 304 vs 100–380 MiB/s acknowledged plus a long drain at 16 MiB). Direct writes are also on the data nodes before the client gets its 200.

**The flusher and crash recovery read headers, not payloads.** Each part has a small header record next to its payload. The flusher's scan, crash recovery at startup, AbortMultipartUpload and ListMultipartUploads read only headers; a payload is loaded just before it is flushed. Earlier builds loaded every buffered part with its payload on every flush tick and at startup, so a backlog larger than the heap made an API node fail to start with `OutOfMemoryError`. Records written by those builds get headers when the store opens.

Knobs:

| Key | Default | Effect |
| --- | --- | --- |
| `shannonstore.api.part.buffer.enabled` | `true` | Master switch. |
| `shannonstore.api.part.buffer.max.bytes` | `262144` (256 KiB) | Largest single-PUT object (stored bytes) that is buffered; larger ones are written straight to the data nodes. Multipart parts are always buffered. `0` = no single PUT is buffered. |
| `shannonstore.api.part.buffer.flush.interval.ms` | `5000` | Flusher cadence. |
| `shannonstore.api.part.buffer.flush.parallelism` | `4` | Objects flushed concurrently (parts of one object stay sequential). |
| `shannonstore.api.part.buffer.rocksdb.path` | `./data/s3-metadata/part-buffer` | Store location. |
| `shannonstore.api.part.buffer.rocksdb.write.buffer.bytes` / `.write.buffer.number` | `67108864` / `4` | Memtable size and count (native memory: at most their product). |
| `shannonstore.api.part.buffer.rocksdb.compression` | `NONE` | The buffered data is ciphertext, which does not compress. |
| `shannonstore.api.part.buffer.rocksdb.blob.files.enabled` / `.min.blob.bytes` / `.blob.file.bytes` | `true` / `65536` / `268435456` | Large values in blob files, so flushes and compactions move references instead of rewriting payloads. |
| `shannonstore.api.part.buffer.peer.fetch.timeout.seconds` | `10` | Timeout when a read or CompleteMultipart needs a part buffered on a peer. |

The buffer is **cluster-aware**: a part uploaded to API node A and a completion call landing on API node B fetches the buffered bytes from A over the internal NIO plane, so SDKs can multipart-upload across a load balancer without sticky sessions.

Parts are also **pre-assigned to EC shard slots** at buffer time. The slot assignment uses the same HRW that final-write placement would, so at completion the per-part shards are already in their final layout — no re-encoding, just promotion from "buffered" to "committed".

**Overwriting a key while it is still buffered: the last PUT wins.** A single PUT goes through the same buffer, and in a bucket without versioning every PUT of a key has the version `null`. When a key is written again before the flusher has drained the first write, the flusher sees that the object's metadata now belongs to the later PUT and drops the earlier bytes instead of writing them into the later PUT's shard slots. (Before 1.0.0's fix, two PUTs of one key within the flush interval could leave the object unreadable, every shard failing its checksum.) `tests/test-overwrite-race-e2e.sh` writes keys twice back to back and checks each reads back as its second body on every API node.

**A GET that cannot be served fails at once.** When fewer shards than the erasure code needs are readable — data nodes unreachable, or shards failing their CRC32C check — the read answers `500 InternalError` as soon as every fetch has returned, instead of waiting out `shannonstore.api.s3.fetch.timeout.seconds`.

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
| S3 request workers | `netty.worker.threads.multiplier × cpus` | One HTTP/1.1 request or HTTP/2 stream each on 8080 (`NioHttpServer`, not Netty). Idle keep-alive connections hold no worker. |
| Fetch pool | `fetch.thread.pool.size` (default `max(16, 4 × cpus)`) | Per-shard NIO reads on the internal plane |
| Chunk pool | `chunk.thread.pool.size` (default `max(16, 4 × cpus)`) | EC encode/decode, KMS unwrap, compression |

The split exists because doubling the wrong pool wastes the resource. Fetch threads spend nearly all their time blocked on sockets; doubling them gives more concurrency at near-zero CPU cost. Chunk threads spend nearly all their time on CPU; doubling them past `availableProcessors()` produces no gain. An operator confused about which pool to scale can read the metrics:

| Metric | Saturation hint |
| --- | --- |
| Fetch-pool acquire timeouts appearing in the API-node logs | fetch pool too small (or pool size per host too small) |
| `system.cpu.usage` pegged at 100% | chunk pool can't process incoming work; PUT/GET latency is CPU-bound |
| Many `NioHttpWorker-8080` threads `RUNNABLE` in a thread dump and request latency rising with concurrency | request workers saturated: raise `netty.worker.threads.multiplier` |
| Growing `TIME_WAIT` sockets on clients / proxy, or established connections on 8080 churning under steady load | connections are not being reused: check client pool sizes and idle timeouts, and the proxy's upstream keep-alive ([HTTP Protocols & Client Tuning](http-protocols.md#client-tuning)) |

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

## Connection reuse and HTTP/2

Many-small-request workloads (lakehouse metadata, Parquet footers, and column chunks) are
dominated by per-request connection cost unless connections are reused. The S3 port keeps
HTTP/1.1 connections alive and serves HTTP/2 (h2c) on the same port. On a 32-thread small-GET
benchmark, reusing connections instead of reconnecting per request raised throughput by 62% and cut
p99 latency from 46 ms to 11 ms (loopback, no TLS; the gap is larger across a network or a TLS
proxy). Make sure the client actually pools:

- nginx: `upstream { keepalive 256; keepalive_requests 10000; keepalive_timeout 60s; }` with `proxy_http_version 1.1` and `proxy_set_header Connection ""`. See [Nginx Reverse Proxy](../operations/nginx-proxy.md).
- s3a / AWS SDK / boto3 pool sizes and idle timeouts: see [HTTP Protocols & Client Tuning](http-protocols.md#client-tuning).
- To multiplex end to end over HTTP/2, put HAProxy (`proto h2`) or Envoy in front instead of nginx.

## Multi-range GETs

`Range: bytes=0-99,200-299` is answered with `206 multipart/byteranges`. Overlapping or adjacent
ranges are merged first. The body is **streamed slice by slice**, with its total length computed up
front, so heap use does not grow with the size of the ranges. A request with more than
`shannonstore.api.s3.max.ranges.per.request` (64) ranges gets the whole object. Readers that
coalesce nearby column chunks into one request benefit, and so do readers that issue separate
range GETs on a reused connection.

## Large-object path

Profiling large GETs and PUTs against MinIO on the same cluster shape (2026-11, see
[HTTP Protocols](http-protocols.md#measured-effect)) found the costs below. Each fix is on by
default, and each is a setting in `shannonstore.properties`.

| Cost | Fix | Setting |
| --- | --- | --- |
| Every GET requested all k+m shards and used the first k to arrive. That meant a third more disk and network I/O, and a Reed-Solomon reconstruction whenever a parity shard won the race. | Healthy reads fetch only the k data shards: no decoding, no parity I/O. Parity is requested when a data shard fails, is missing, or is slower than the hedge delay. | `shannonstore.api.s3.ec.parity.hedge.ms` (`2000`) |
| AES-GCM (object encryption at rest) through the JDK provider: about 95 MiB/s per core on aarch64. Encrypting or decrypting a 16 MiB object took about 170 ms of CPU. | The native Amazon Corretto Crypto Provider (AWS-LC), about 5.5 GB/s. It implements the same standard algorithm, so data written with either provider is readable with the other. It falls back to the JDK when the native library cannot load. | `shannonstore.crypto.native.provider.enabled` (`true`) |
| Internal shard transfers allocated a direct buffer per message, opened two selectors per RPC, and busy-spun on a full socket while copying the whole remaining shard on every attempt. | One selector per connection, bounded read/write slices, waiting for the socket instead of spinning, and no extra copy of the payload. | `shannonstore.network.nio.io.slice.bytes` (`262144`) |
| Shards (AES ciphertext) were Snappy-compressed for the network, which costs CPU and copies for no gain. | A sample is compressed first, and the payload is sent as-is when it does not shrink. | `shannonstore.network.compression.sample.bytes`, `.min.saving.percent` |
| EC encode called a GF(2⁸) multiply per byte per shard. | A table-driven encoder with sequential access per shard. Its output is byte-identical (verified against the original). | — |
| Request bodies were read 8 KiB at a time into a side buffer, and aws-chunked headers one byte per socket read. | Bodies are read straight into the destination array, and the aws-chunked decoder is buffered. | `shannonstore.network.http.body.read.buffer.size` (`262144`) |
| Data nodes read shard files through a temporary 4 MiB native buffer that was allocated and zero-filled on every read. | Shard files are read in bounded slices. | (same slice setting) |

### Block streaming (format 2)

Encrypted large objects were still slow under concurrency, because a PUT or GET held the whole
part in memory and processed it in sequence: buffer it, encrypt it, erasure-code it, send the shards,
and the reverse on read. Sixteen concurrent 16 MiB GETs kept several hundred MiB of short-lived
arrays alive, and the first byte reached the client only after the last shard had been decoded.

Since 2026-11-07, a single PUT larger than `shannonstore.api.part.buffer.max.bytes` is stored in
**format 2** (as MinIO does):

- The body is processed in 1 MiB blocks as it arrives. Each block is encrypted with AES-GCM on its
  own, erasure-coded, and its pieces are appended to the k+m shard files on the data nodes while the
  next block is read from the client.
- A GET reads the pieces of a few blocks ahead (`stream.read.ahead.blocks`), decodes each block, and
  starts sending after the first block. A range GET reads only the blocks it covers.
- Memory per request is bounded by the block size × the read-ahead and in-flight windows, not by the
  object size. `stream.max.concurrent` caps the streams an API node serves at once. A request over
  the cap gets `503 SlowDown`, which S3 SDKs retry with backoff.

Multipart parts, objects up to 256 KiB, compressed objects and SSE-C objects keep the earlier layout
(format 1). Objects written by older builds stay readable, and the two formats coexist in one bucket.
The shard layout is described in [Erasure Coding](ec.md#block-streaming-format-2).

| Setting | Default | Meaning |
| --- | --- | --- |
| `shannonstore.api.s3.stream.format.enabled` | `true` | Store large single PUTs in format 2. `false` writes format 1; both are always readable. |
| `shannonstore.api.s3.stream.block.bytes` | `1048576` | Plaintext block size. Larger blocks mean fewer RPCs and more memory per stream. |
| `shannonstore.api.s3.stream.max.concurrent` | `64` | Format-2 PUTs and GETs one API node runs at once; more get `503 SlowDown`. |
| `shannonstore.api.s3.stream.read.ahead.blocks` | `4` | Blocks a GET fetches ahead of the one being sent. |
| `shannonstore.api.s3.stream.shard.inflight` | `2` | Pieces a PUT may have in flight to one shard. |

!!! warning "Set a heap limit"
    Give API and data nodes an explicit `-Xmx` (in `conf/jvm.conf` or on the command line) and a
    container memory limit. Without `-Xmx`, every JVM may grow its heap to 25% of the machine. Five
    of them on one host can exceed its memory, and the kernel's OOM killer then stops one under
    load. The docker compose test cluster uses `-Xmx1536m` for API nodes and `-Xmx1g` for data
    nodes.

## What's deliberately not optimised

One path takes simple-and-correct over fast-but-complex:

- **Coalesced metadata writes**. Each IAM mutation immediately fans out a snapshot rather than batching multiple mutations into one push. Batching would lower CPU but increases the leader→follower convergence window; the convergence guarantee is more valuable than the savings.

## See also

- [Erasure Coding](ec.md) — the encoding step the chunk pool drives.
- [Network & Internal Communication](network.md) — the connection pool and threading model in more depth.
- [HTTP Protocols & Client Tuning](http-protocols.md) — keep-alive, HTTP/2, and client connection pools.
- [Encryption & Key Management](kms.md) — the KMS unwrap that runs on the chunk pool.
- [Disk Repair Service](disk-repair.md) — the recipient of background-write failures from quorum writes.
- [Configuration Management](configuration.md) — the full set of knobs mentioned here.
- [Monitoring & Metrics](monitoring.md) — the gauges that tell an operator which knob to tune.
