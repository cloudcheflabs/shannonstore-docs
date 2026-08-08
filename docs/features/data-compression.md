# Data Compression

ShannonStore compresses data at three independent layers, each with its own configuration: object-level (the payload as the client wrote it), network-level (inter-node payloads on port 9090), and metadata-level (RocksDB values that hold object indices). Each layer can be tuned in isolation, so an operator can leave object compression off when storing already-compressed binaries while still compressing the metadata that describes them.

```text
   Client PUT       ┌─────────────────────────────────┐
       │            │  Object-level compression       │
       ▼            │  ─ MIME match? compress payload │
   ┌────────┐       │  ─ algorithm: SNAPPY (default)  │
   │  API   │       └─────────────────────────────────┘
   │  node  │            │
   │        │            ▼
   │        │       ┌─────────────────────────────────┐
   │        │       │  EC + KMS encrypt + …            │
   │        │       └─────────────────────────────────┘
   │        │            │
   │        │            ▼
   │        │       ┌─────────────────────────────────┐
   │        │       │  Network-level compression       │
   │        │       │  ─ >1 KiB? compress              │
   │        │       │  ─ algorithm: SNAPPY (default)  │
   │        │       └─────────────────────────────────┘
   └────────┘            │
                         ▼
                    Data node disk
                    (chunk file — already compressed
                     payload, optional data-node-side
                     compression on top, normally NONE)
```

The three layers compose multiplicatively for CPU but **not** for storage savings — only the lowest-level "wins" on bytes. Picking the right layer for the workload matters.

## Object-level compression

Applied at the API node *before* erasure coding and KMS encryption. The compressed payload is what gets sharded and persisted, so the storage savings translate directly to disk bytes.

| Key | Default | Effect |
| --- | --- | --- |
| `shannonstore.api.s3.object.compression.type` | `SNAPPY` | Algorithm: `NONE` / `SNAPPY` / `GZIP` / `ZSTD`. |
| `shannonstore.api.s3.object.compression.mime.types` | `text/*,application/json,application/xml` | Globs — only matching `Content-Type` headers compress. |

### MIME-based selection

The configured globs gate the compression decision per-object:

| `Content-Type` | Match? | Action |
| --- | --- | --- |
| `text/plain` | ✓ matches `text/*` | compress |
| `application/json` | ✓ exact match | compress |
| `application/xml` | ✓ exact match | compress |
| `application/parquet` | ✗ | store as-is |
| `image/png`, `image/jpeg` | ✗ | store as-is — already compressed by the format |
| `video/mp4`, `application/zstd` | ✗ | store as-is |
| `application/octet-stream` | ✗ | store as-is (conservative — the SDK has lost type info) |

The decision is per-object, made once at PUT time, and recorded in the object's metadata. A subsequent GET re-reads the metadata and runs the matching decompression — there's no client-visible Content-Encoding negotiation. This is the AWS-S3-spec-faithful path: the bytes the client gets back are byte-identical to the bytes they uploaded.

### Algorithm trade-offs

| Algorithm | Throughput | Ratio (text) | When to pick |
| --- | --- | --- | --- |
| `NONE` | n/a | 1× | Storing already-compressed formats (Parquet/ORC/PNG/MP4); CPU-bound API nodes. |
| `SNAPPY` (default) | very fast | 2–3× on JSON/log | Default sweet spot — minimal CPU cost, real savings. |
| `GZIP` | moderate | 3–6× on JSON/log | Maximize savings on highly redundant data, accept the CPU hit. |
| `ZSTD` | fast | 3–5× | Modern Zstandard — better ratio than SNAPPY at roughly the same throughput. The right pick for clusters that aren't ancient JVM. |

ZSTD is the recommended algorithm for new deployments. SNAPPY remains the default to preserve backwards compatibility with older releases that didn't carry the ZSTD native library.

### What compression buys you

For a typical analytics workload (Iceberg + Parquet + JSON event logs):

- Parquet objects: 0% savings (already compressed by Parquet's columnar encoders).
- Raw JSON logs: 75–85% savings with ZSTD.
- CSV exports: 70–80% savings with ZSTD.
- Application checkpoints (Spark / Flink state): varies — usually 30–50%.

For mixed media (image + video heavy buckets), object-level compression is a near-zero win and the CPU cost matters more — leave it on `NONE` for those buckets, or rely on the MIME filter to skip them automatically.

## Network-level compression

Applied to **every** inter-node payload above a threshold, regardless of MIME type. Distinct from object-level: this layer compresses shard payloads in flight between API nodes and data nodes, never on disk.

| Key | Default | Effect |
| --- | --- | --- |
| `shannonstore.network.compression.type` | `SNAPPY` | Algorithm. |
| `shannonstore.network.compression.threshold` | `1024` | Bytes — below this skip compression. |

Why threshold-gated? Heartbeat / membership messages are tens of bytes; compressing them costs more in latency than it saves in bandwidth. Shard payloads are tens of MiBs; compressing them halves the wire bytes between nodes at a small CPU cost on each end. The default 1 KiB threshold cleanly divides the two.

The wire format includes the algorithm in a single byte at the head of the payload, so a rolling change of `shannonstore.network.compression.type` doesn't require coordinated restarts — a peer running the old algorithm decompresses the next message correctly without configuration drift.

The network layer **does not** know whether the payload was already object-compressed. A doubly-compressed payload sees a small CPU hit on both sides for nearly zero additional saving — the gzip / zstd / snappy header still adds overhead. Operators who turn object-level compression on can lower the network-level threshold or disable network compression entirely if the metric `shannonstore_network_bytes_in_total` doesn't move when network compression flips on. The two layers are independent precisely so this tuning is possible.

## Metadata-level compression

The object index in RocksDB stores one entry per `(bucket, key, version)`. Compressing the index values trades CPU at every metadata read/write for a smaller RocksDB footprint:

| Key | Default | Effect |
| --- | --- | --- |
| `shannonstore.api.index.rocksdb.value.compression` | `false` | Master switch. |
| `shannonstore.api.index.rocksdb.value.compression.type` | `SNAPPY` | Algorithm when enabled. |

The metadata blob per object is small (a few hundred bytes) but contains a lot of redundant strings (bucket name, key path, node addresses). With compression enabled the RocksDB footprint shrinks 50–70% on a typical install. The trade-off is a SNAPPY round-trip on every metadata read; on a hot read path this is unmeasurable but on a metadata-heavy workload (ListObjects across millions of keys) it nudges latency up.

Recommendation: leave **off** by default; turn on for clusters that hit the RocksDB capacity wall before they hit the read-latency wall.

## Data-node-side chunk compression

A fourth, defense-in-depth knob on the data node:

| Key | Default | Effect |
| --- | --- | --- |
| `shannonstore.data.storage.chunk.compression.type` | `NONE` | Per-chunk compression on the data-node disk. |

This is intentionally `NONE` by default. Object-level compression already handles the storage savings — adding the data-node layer on top wastes CPU for no incremental ratio. The knob exists for clusters that need a second compression pass at the chunk level for compliance or operational reasons (e.g. older data ingested without object-level compression and an operator who would rather compress in place than rewrite the objects).

## Choosing the right layer for the workload

| Workload | Object | Network | Metadata | Data-node chunk |
| --- | --- | --- | --- | --- |
| Iceberg + Parquet | NONE | SNAPPY | off | NONE |
| Raw JSON logs | ZSTD | SNAPPY | off | NONE |
| Mixed images / videos | NONE (MIME filter skips them anyway) | SNAPPY | off | NONE |
| Backups / archives | ZSTD | SNAPPY | on | NONE |
| Time-series / metrics | ZSTD | SNAPPY | on | NONE |

The default config (SNAPPY object + SNAPPY network + off metadata + NONE chunk) is a reasonable middle ground for the analytics workload that dominates ShannonStore deployments today.

## Diagnostics

There are currently no dedicated Prometheus counters for compression ratio or
skip-rate by layer — grepping the whole codebase for compression-related metric
registrations (`Counter.builder`, `Gauge.builder`, etc.) turns up none. See
[Monitoring & Metrics](monitoring.md) for the actual (much smaller) set of exported
metrics. Verifying compression effectiveness today means comparing object size on
disk to the client-reported `Content-Length`, not reading a ratio off a dashboard.

## See also

- [Erasure Coding](ec.md) — runs *after* object compression on the same payload.
- [Encryption & Key Management](kms.md) — runs *after* compression so the encryption layer never sees compressible patterns.
- [Network & Internal Communication](network.md) — where the network-compression knob lands.
- [Configuration Management](configuration.md) — the resolution chain that produces these keys.
- [Monitoring & Metrics](monitoring.md) — the Prometheus counters that quantify the savings.
