# Configuration Management

ShannonStore separates *what* the cluster does from *how* it's tuned. Every node — API or data — derives its full runtime configuration from a small, well-defined resolution chain, so the same artifact runs unchanged through CI, staging, and prod. The same `ConfigLoader` powers both `ApiConfig` and `DataConfig`; everything below applies to both node types.

## Resolution chain

A configuration key is looked up in this strict order, first hit wins:

```
   1.  -D<key>=<value>          ← JVM system property  (highest priority)
   2.  <KEY>=<value>            ← environment variable (key uppercased, dots → underscores)
   3.  external config file     ← path from -Dshannonstore.config.file=...
   4.  classpath conf/api.properties / conf/data.properties  ← defaults
```

The chain is short on purpose. An operator deploying with a Helm chart sets system properties via JVM flags; a developer iterating locally edits the external file; a container image bakes safe defaults into the classpath. No layer can silently undo a higher-priority value — the resolution stops the moment a key resolves.

A typical production launcher script:

```bash
# Image baked with conf/api.properties as defaults.
exec java \
  -Dshannonstore.config.file=/etc/shannonstore/site.properties \
  -Dshannonstore.api.s3.ec.data.shards=8 \          # override one knob via flag
  -Dshannonstore.api.s3.ec.parity.shards=4 \
  -DSHANNONSTORE_MASTER_KEY="$KMS_FROM_SECRET" \
  -jar shannonstore-api.jar
```

The environment-variable form is what container schedulers prefer:

```yaml
env:
  - name: SHANNONSTORE_API_S3_EC_DATA_SHARDS
    value: "8"
  - name: SHANNONSTORE_API_S3_EC_PARITY_SHARDS
    value: "4"
  - name: SHANNONSTORE_MASTER_KEY
    valueFrom: { secretKeyRef: { name: shannonstore-kms, key: master } }
```

The same key works either way. ShannonStore's loader uppercases dots → underscores so an operator can pick whichever shape their orchestrator makes ergonomic.

## Variable substitution

Values may reference other keys with the `${…}` form. Substitution happens once at start, against the already-resolved key set:

```properties
shannonstore.data.root              = /mnt/shannon
shannonstore.data.storage.dirs      = ${shannonstore.data.root}/disk-1,${shannonstore.data.root}/disk-2
shannonstore.data.kms.rocksdb.path  = ${shannonstore.data.root}/kms-rocksdb
shannonstore.log.path               = ${shannonstore.data.root}/logs
```

This is the DRY mechanism for keeping per-disk paths and log paths anchored to a single root that changes between environments. Cycles in the substitution graph are rejected at start with a clear log line.

## Key surface — what's actually tunable

Over 150 keys cover every dimension of the cluster. The most-encountered groupings:

### Cluster connectivity

| Key | Default | Purpose |
| --- | --- | --- |
| `shannonstore.zk.connect` | `localhost:2181` | ZooKeeper coordinate for leader election / membership. |
| `shannonstore.advertised.host` | `localhost` | Hostname peers and clients reach this node by. |
| `shannonstore.api.s3.port` | `8080` | S3 wire endpoint. |
| `shannonstore.api.admin.port` | `8888` | Admin REST + Admin UI. |
| `shannonstore.nio.port` | `9090` | Internal NIO channel for inter-node RPC. |
| `shannonstore.api.zk.discovery.path` | `/s3/discovery/api` | ZooKeeper path for API-node membership advertisement. |
| `shannonstore.api.zk.leader.path` | `/s3/masters/leader` | ZooKeeper path for the API leader election. |

### Storage layout

| Key | Default | Purpose |
| --- | --- | --- |
| `shannonstore.api.s3.ec.data.shards` | `4` | Reed-Solomon k. |
| `shannonstore.api.s3.ec.parity.shards` | `2` | Reed-Solomon m. |
| `shannonstore.api.iam.rocksdb.dir` | `./data/s3-metadata/iam` | IAM/bucket state RocksDB. |
| `shannonstore.data.storage.dirs` | `./data/s3-data-data` | Comma-separated storage roots on data nodes. |
| `shannonstore.data.kms.rocksdb.path` | `./data/shannon-data-kms-rocksdb` | Wrapped DEK store on the data node. |

### Security

| Key | Default | Purpose |
| --- | --- | --- |
| `SHANNONSTORE_MASTER_KEY` | _required_ | KEK for envelope encryption. Process refuses to start without it. |
| `shannonstore.api.s3.object.encryption.type` | `AES` | Object-level encryption algorithm. |
| `shannonstore.api.sts.default.session.duration.seconds` | — | Cap on STS `AssumeRole` lifetimes. |

### Performance tuning

| Key | Default | Purpose |
| --- | --- | --- |
| `shannonstore.api.fetch.thread.pool.size` | `2 × cpus` | NIO fetch pool for parallel shard reads. |
| `shannonstore.api.chunk.thread.pool.size` | `2 × cpus` | Chunk encode / decode pool. |
| `shannonstore.api.connection.pool.size.per.host` | `64` | Idle NIO connections retained per peer. |
| `shannonstore.api.connection.pool.acquire.timeout.seconds` | `30` | Blocking acquire on a busy pool. |
| `shannonstore.api.connection.pool.warmup.enabled` | `true` | Pre-open connections at startup. |
| `shannonstore.api.prefetch.queue.size` | `8` | Async download lookahead. |
| `shannonstore.api.prefetch.initial.count` | `8` | Initial shards prefetched on GET. |
| `shannonstore.api.s3.small.object.threshold` | `67108864` (64 MiB) | Cut-off between buffered and streaming GET. |
| `shannonstore.api.s3.circular.buffer.size` | `67108864` | Streaming GET ring-buffer per request. |
| `shannonstore.api.s3.chunked.stream.size` | `1048576` | Streaming chunk size on the wire. |
| `shannonstore.api.s3.fetch.timeout.seconds` | `60` | Per-shard fetch deadline. |
| `shannonstore.api.netty.worker.threads.multiplier` | `4` | Multiplier on `availableProcessors()` for Netty worker pool. |

### Compression

| Key | Default | Purpose |
| --- | --- | --- |
| `shannonstore.api.s3.object.compression.type` | `SNAPPY` | Per-object compression: `NONE` / `SNAPPY` / `GZIP` / `ZSTD`. |
| `shannonstore.api.s3.object.compression.mime.types` | `text/*,application/json,application/xml` | MIME glob — only matching content types are compressed. |
| `shannonstore.network.compression.type` | `SNAPPY` | Inter-node payload compression. |
| `shannonstore.network.compression.threshold` | `1024` | Bytes below this skip compression. |
| `shannonstore.api.index.rocksdb.value.compression` | `false` | Compress metadata RocksDB values. |
| `shannonstore.api.index.rocksdb.value.compression.type` | `SNAPPY` | Algorithm when enabled. |

### Metrics & observability

| Key | Default | Purpose |
| --- | --- | --- |
| `shannonstore.api.prometheus.enabled` | `true` | Expose `/metrics` for scraping. |
| `shannonstore.api.metrics.collection.interval.seconds` | `60` | System metrics tick. |
| `shannonstore.api.metrics.retention.days` | `30` | History kept in the internal store. |
| `shannonstore.api.metrics.retention.cleanup.interval.hours` | `12` | Sweep cadence. |
| `shannonstore.api.history.retention.days` | `365` | Operation-history retention. |

### Data-node specifics

| Key | Default | Purpose |
| --- | --- | --- |
| `shannonstore.data.storage.dirs` | `./data/s3-data-data` | One or more storage roots (typically one per disk). |
| `shannonstore.data.storage.chunk.compression.type` | `NONE` | On-disk compression (independent of object-level). |
| `shannonstore.data.storage.chunk.encryption.type` | `NONE` | Defense-in-depth at the data-node disk layer. |
| `shannonstore.data.thread.pool.compute.size` | `512` | CPU-bound pool size. |
| `shannonstore.data.disk.info.refresh.interval.seconds` | `60` | How often disk-health stats publish to the API tier. |
| `shannonstore.data.kms.pull.max.attempts` | `60` | Retry budget when fetching KMS at startup. |
| `shannonstore.data.kms.pull.rpc.timeout.seconds` | `5` | Per-attempt deadline. |
| `shannonstore.data.startup.leader.ready.poll.sleep.ms` | `500` | Sleep between leader-readiness polls at boot. |

A full enumeration lives in `conf/api.properties` and `conf/data.properties` inside the release tarball — those files are the canonical reference for the complete set, with comments next to each key.

## Hot-reload vs restart

Most keys take effect at process start. A small subset is mutable at runtime through the Admin REST:

| Key area | Mutable at runtime? | How |
| --- | --- | --- |
| Maintenance mode | yes | `POST /admin/maintenance` |
| Bitrot scrubber enable / pause | yes | Admin UI toggle (per-node) |
| Disk-repair concurrency / throttle | yes | Admin UI knob |
| Metrics retention | yes | `PATCH /admin/config/retention` |
| EC / KMS / port / ZK topology | no | restart required |

The static-vs-mutable split is intentional. A live cluster's storage layout and addressing should not change without an explicit deploy; everything that's a *policy* (when to scrub, when to throttle) is meant to respond to operator judgement without a restart.

## Validation and fail-closed defaults

The loader validates every parsed value at start. Sensitive failure modes:

- Missing `SHANNONSTORE_MASTER_KEY` → process refuses to launch (see [KMS](kms.md)).
- Unparseable integer / boolean → process refuses to launch with the offending key in the log.
- Substitution cycle → process refuses to launch with the cycle path printed.
- Out-of-range value (negative port, EC k+m > cluster nodes) → process logs a warning and uses the safe default.

The "fail closed" choice is consistent with the rest of the security model: a misconfigured deployment never silently runs with weaker guarantees than the operator believed they configured.

## See also

- [Installation](../installation/installation.md) — the bootstrap path that consumes these keys.
- [Performance Optimization](performance.md) — the runtime knobs that shape latency and throughput.
- [Encryption & Key Management](kms.md) — the master-key requirement and rotation flow.
- [Monitoring & Metrics](monitoring.md) — the metrics surfaces produced by the keys in the metrics section.
