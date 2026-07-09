# Configuration Management

ShannonStore separates *what* the cluster does from *how* it's tuned. Every node — API or Data — derives its full runtime configuration from a single, well-defined resolution chain, so the same artifact runs unchanged through CI, staging, and production. One `ConfigLoader` powers both the API Server and the Data Node; everything below applies to both node types.

The canonical source of every key — with an inline comment next to each one — is `conf/shannonstore.properties` in the release tarball. The tables on this page mirror that file section by section.

## Resolution chain

A configuration key is looked up in this strict order, **first hit wins**:

```
   1.  -D<key>=<value>            ← JVM system property        (highest priority)
   2.  <KEY>=<value>              ← environment variable       (key uppercased, dots → underscores)
   3.  external properties file   ← path from -Dshannonstore.config.file=... (or SHANNONSTORE_CONFIG_FILE)
   4.  classpath shannonstore.properties  ← bundled defaults  (lowest priority)
```

The system property and environment variable are consulted at every lookup, so they override the file-supplied value for that key. The external properties file (if supplied) is merged *over* the bundled `shannonstore.properties` at startup, so it overrides only the classpath defaults. No layer can silently undo a higher-priority value.

### Environment-variable naming convention

Any key can be supplied as an environment variable: take the property key, replace every dot with an underscore, and uppercase it. For example:

| Property key | Environment variable |
| --- | --- |
| `shannonstore.api.s3.port` | `SHANNONSTORE_API_S3_PORT` |
| `shannonstore.zk.connect` | `SHANNONSTORE_ZK_CONNECT` |
| `shannonstore.api.s3.ec.data.shards` | `SHANNONSTORE_API_S3_EC_DATA_SHARDS` |

A typical production launcher mixes the forms — file for the bulk, a `-D` or env var for the few per-host overrides:

```bash
exec java \
  -Dshannonstore.config.file=/etc/shannonstore/site.properties \
  -Dshannonstore.api.s3.ec.data.shards=8 \
  -Dshannonstore.api.s3.ec.parity.shards=4 \
  -jar shannonstore-api.jar
```

The container-scheduler form is equivalent:

```yaml
env:
  - name: SHANNONSTORE_API_S3_EC_DATA_SHARDS
    value: "8"
  - name: SHANNONSTORE_MASTER_KEY
    valueFrom: { secretKeyRef: { name: shannonstore-kms, key: master } }
```

## Variable substitution

Values may reference other keys with the `${…}` form. References are resolved at lookup time using the same precedence (system property → environment variable → properties file). Most path keys default to a value anchored under `shannonstore.base.data.dir`, so overriding that single key relocates a node's entire on-disk state:

```properties
shannonstore.base.data.dir          = /mnt/shannon
# the following all resolve under /mnt/shannon automatically:
shannonstore.data.storage.dirs      = ${shannonstore.base.data.dir}/s3-data
shannonstore.data.kms.rocksdb.path  = ${shannonstore.base.data.dir}/shannon-data-kms-rocksdb
shannonstore.log.path               = ${shannonstore.base.data.dir}/logs
```

An unresolved reference is left as the literal `${…}` text rather than failing.

---

## 1. Common Cluster Configuration

Applies to both API and Data nodes.

| Property | Default | Description |
| --- | --- | --- |
| `shannonstore.advertised.host` | `localhost` | Hostname or IP this node publishes (via ZooKeeper) for peers to reach it on the NIO port. If left `localhost`/empty/unset, the node auto-resolves to its hostname (falling back to IP). Set explicitly in multi-host deployments. |
| `shannonstore.nio.port` | `9090` | TCP port for the internal NIO RPC server on both API and Data nodes — all inter-node traffic (chunk transfer, metadata RPC, KMS pull, backup fan-out) uses it. Must be open between all nodes. |
| `shannonstore.zk.connect` | `localhost:2181` | ZooKeeper connection string (`host1:port1,host2:port2`) for cluster coordination, leader election, and node discovery. Every node must point at the same ensemble. |
| `shannonstore.base.data.dir` | `./data` | Base directory for all on-disk state. Interpolated into nearly every other path key, so overriding it relocates the node's entire state. Relative paths resolve against the process working directory. |
| `shannonstore.zk.retry.base.sleep.ms` | `1000` | Base sleep (ms) for the Curator `ExponentialBackoffRetry` policy on the ZooKeeper client. |
| `shannonstore.zk.retry.max.retries` | `3` | Maximum retry attempts in the ZooKeeper client's backoff policy before abandoning an operation. |
| `shannonstore.network.compression.type` | `SNAPPY` | Compression for internal NIO message payloads between nodes. Supported: `NONE`, `SNAPPY`. |
| `shannonstore.network.compression.threshold` | `1024` | Minimum payload size (bytes) below which network compression is skipped. Only consulted when compression type is not `NONE`. |
| `shannonstore.network.direct.buffer.enabled` | `true` | Whether the NIO layer allocates direct (off-heap) ByteBuffers for socket I/O, avoiding a heap-to-native copy at the cost of off-heap memory. |
| `shannonstore.network.socket.send.buffer` | `2097152` (2 MiB) | TCP send buffer (`SO_SNDBUF`, bytes) on inter-node sockets. |
| `shannonstore.network.socket.recv.buffer` | `2097152` (2 MiB) | TCP receive buffer (`SO_RCVBUF`, bytes) on inter-node sockets. |
| `shannonstore.kms.type` | `cluster` | KMS provider. `cluster` is the built-in distributed KMS (KEKs envelope-encrypted in a local RocksDB keystore, synced from the leader). |
| `shannonstore.kms.master.key.env` | `SHANNONSTORE_MASTER_KEY` | Name of the OS environment variable holding the cluster Master Encryption Key. Run through PBKDF2 to derive the root key that wraps every KEK; on API nodes it also seeds the Admin JWT signing secret. Must be identical on all nodes. |
| `shannonstore.kms.pbkdf2.iterations` | `200000` | PBKDF2 iteration count used to stretch the master passphrase into the root key. Must be identical across all nodes. |
| `shannonstore.kms.default.key.id` | `default` | Identifier of the default KEK used to envelope-encrypt object data and metadata blobs when no key is specified. |
| `shannonstore.kms.sse.s3.key.id` | `default-sse-s3-key` | Identifier of the KEK used for objects written via the S3 SSE-S3 path (`x-amz-server-side-encryption: AES256`). |

---

## 2. API Node Configuration

API node only.

| Property | Default | Description |
| --- | --- | --- |
| `shannonstore.api.s3.port` | `8080` | TCP port for the public S3-compatible HTTP API (PutObject/GetObject/ListObjects, SigV4). The endpoint S3 clients connect to. |
| `shannonstore.api.admin.port` | `8888` | TCP port for the Admin Web Console and admin REST API (JWT-protected). |
| `shannonstore.api.iam.rocksdb.dir` | `${shannonstore.base.data.dir}/s3-metadata/iam` | RocksDB directory for IAM state (users, access keys, policies). The leader is authoritative; followers pull and replay it. |
| `shannonstore.api.admin.ui.static.path` | `admin-ui` | Path to the Admin Web Console's static front-end assets. **Caveat:** the in-code default is `shannonstore-admin-ui/dist`; the bundled file sets `admin-ui` to match the packaged layout. |
| `shannonstore.api.index.rocksdb.path` | `${shannonstore.base.data.dir}/s3-metadata/rocksdb` | Base directory of the object-metadata index RocksDB. When sharding is enabled and `metadata.rocksdb.dirs` is empty, per-shard sub-stores live here. |
| `shannonstore.api.kms.rocksdb.path` | `${shannonstore.base.data.dir}/s3-metadata/kms-rocksdb` | Directory of the API node's KMS keystore RocksDB holding envelope-encrypted KEKs. |
| `shannonstore.api.index.rocksdb.value.compression` | `false` | Whether RocksDB value compression is enabled on the metadata index store. |
| `shannonstore.api.index.rocksdb.value.compression.type` | `SNAPPY` | Codec used only when value compression is enabled. Supported: `NONE`, `SNAPPY`, `GZIP`, `ZSTD`. |
| `shannonstore.api.index.cache.ttl.seconds` | `300` | TTL (seconds) for entries in the in-memory metadata cache over the index RocksDB. |
| `shannonstore.api.index.cache.cleanup.interval.seconds` | `60` | Period (seconds) of the background sweep evicting expired metadata-cache entries. |
| `shannonstore.api.fetch.thread.pool.size` | `48` | Thread pool for fetching data chunks from Data nodes (I/O-bound). In-code default is `max(16, cores × 4)`. |
| `shannonstore.api.chunk.thread.pool.size` | `48` | Thread pool for uploading data chunks to Data nodes (symmetric with fetch). |
| `shannonstore.api.prefetch.queue.size` | `8` | Max object chunks kept in-flight (read-ahead) in the streaming-GET prefetch pipeline. |
| `shannonstore.api.prefetch.initial.count` | `8` | Chunks fetched eagerly at the start of a streaming read before the steady-state window kicks in. |
| `shannonstore.api.thread.pool.task.executor.size` | `48` | General-purpose async executor for S3 request handling and background work. In-code default `max(16, cores × 4)`. |
| `shannonstore.api.metadata.persistence.thread.pool.size` | `4` | Background threads flushing dirty metadata down to the index RocksDB. **Caveat:** in-code default is `1`; the file sets `4` for higher write throughput. |
| `shannonstore.api.netty.worker.threads.multiplier` | `4` | Multiplier for sizing Netty/HTTP worker event-loop threads: `cores × multiplier`. |
| `shannonstore.api.connection.pool.size.per.host` | `64` | Max persistent (keep-alive) connections retained in the API→Data-node pool per remote host. |
| `shannonstore.api.connection.pool.acquire.timeout.seconds` | `30` | Max time (seconds) a caller blocks waiting to borrow a pooled connection before failing. |
| `shannonstore.api.connection.pool.warmup.enabled` | `true` | Eagerly open (warm) connections to Data nodes at startup so early requests avoid setup latency. |
| `shannonstore.api.s3.small.object.threshold` | `67108864` (64 MiB) | Objects ≤ this content-length take the single-shot in-memory path; larger ones take the streaming/multi-segment path. |
| `shannonstore.api.s3.multipart.segment.size` | `16777216` (16 MiB) | Segment size for the multi-segment upload path on large objects. Smaller values reduce range amplification on later Range GETs. |
| `shannonstore.api.s3.circular.buffer.size` | `134217728` (128 MiB) | Capacity (bytes) of the `CircularBufferInputStream` used to stream very large objects out to the client. **Caveat:** in-code default is 64 MiB; the file sets 128 MiB. |
| `shannonstore.api.s3.chunked.stream.size` | `1048576` (1 MiB) | Heap buffer size per read/copy step of chunked HTTP transfer on S3 GET/PUT. |
| `shannonstore.api.s3.fetch.timeout.seconds` | `120` | Max time (seconds) the API node waits for a Data node to return a chunk. **Caveat:** in-code default is 60; the file sets 120. |
| `shannonstore.api.network.http.max.content.length` | `268435456` (256 MiB) | Maximum permitted HTTP request body length (bytes); effectively caps a single non-streaming PutObject body. |
| `shannonstore.api.s3.object.encryption.type` | `AES` | Encryption of object data at rest. `AES` = envelope-encrypt chunks with a KEK; `NONE` = plaintext. |
| `shannonstore.api.s3.object.versioning.enabled` | `false` | Enable S3 bucket object versioning (keep prior versions instead of overwriting). |
| `shannonstore.api.s3.object.compression.type` | `SNAPPY` | Codec applied to object data before storage (only for MIME types in the allowlist below). Supported: `NONE`, `SNAPPY`, `GZIP`, `ZSTD`. |
| `shannonstore.api.s3.object.compression.mime.types` | `text/*,application/json,application/xml` | Comma-separated MIME allowlist eligible for compression; subtype wildcards allowed (e.g. `text/*`). |
| `shannonstore.api.s3.ec.data.shards` | `4` | Erasure Coding data-shard count *k*: each object is split into *k* data shards. |
| `shannonstore.api.s3.ec.parity.shards` | `2` | Erasure Coding parity-shard count *m*: redundant shards per object; tolerates up to *m* simultaneous failures (4+2 scheme by default). |
| `shannonstore.api.zk.discovery.path` | `/s3/discovery/api` | ZooKeeper znode path where API nodes register ephemeral discovery entries. |
| `shannonstore.api.zk.leader.path` | `/s3/masters/leader` | ZooKeeper base path for the `LeaderLatch` in API leader election; also read by Data nodes to locate the leader. |
| `shannonstore.api.zk.coordinator.leader.id.path` | `/s3/coordinator-leader-id` | Persistent znode recording the incumbent leader's nodeId; source of truth for sticky-leader election across restarts. |
| `shannonstore.api.cluster.leader.election.timeout.ms` | `30000` | Max time (ms) to wait for a leader to be elected before continuing startup. |
| `shannonstore.api.cluster.leader.deference.window.ms` | `3000` | Sticky-leader deference window (ms): a non-incumbent node waits this long for the prior incumbent to reclaim the latch. 2000–5000 ms is a reasonable range. |
| `shannonstore.api.cluster.sync.retry.count` | `3` | Retry attempts for inter-API-node cluster sync operations. |
| `shannonstore.api.cluster.sync.retry.delay.ms` | `1000` | Delay (ms) between cluster-sync retry attempts. |
| `shannonstore.api.admin.jwt.access.token.expiration.ms` | `900000` (15 min) | Lifetime of an issued Admin API JWT access token. |
| `shannonstore.api.admin.jwt.refresh.token.expiration.ms` | `86400000` (24 h) | Lifetime of an Admin refresh token (mints new access tokens without re-login). |
| `shannonstore.api.admin.jwt.cleanup.interval.ms` | `10800000` (3 h) | Period of the background sweep purging expired/revoked refresh tokens. |

### Part Buffer (write-back cache)

| Property | Default | Description |
| --- | --- | --- |
| `shannonstore.api.part.buffer.enabled` | `true` | When true, multipart-upload parts are first written to a fast local RocksDB-backed buffer and asynchronously flushed to Data nodes, accelerating ingest. |
| `shannonstore.api.part.buffer.rocksdb.path` | `${shannonstore.base.data.dir}/s3-metadata/part-buffer` | RocksDB directory backing the part buffer. |
| `shannonstore.api.part.buffer.flush.interval.ms` | `5000` | Interval (ms) at which the background flusher drains buffered parts to Data nodes. |
| `shannonstore.api.part.buffer.memory.max.bytes` | `2147483648` (2 GiB) | Soft cap (bytes) on the in-memory (L1) part-buffer footprint before parts are evicted. |

### Advanced RocksDB tuning (metadata index store)

| Property | Default | Description |
| --- | --- | --- |
| `shannonstore.api.rocksdb.max.open.files` | `256` | Max open SST file descriptors RocksDB caches. `-1` = unlimited. |
| `shannonstore.api.rocksdb.write.buffer.size` | `16777216` (16 MiB) | RocksDB write buffer per memtable. Smaller = less RSS; larger = fewer compactions. |
| `shannonstore.api.rocksdb.max.write.buffer.number` | `2` | Max memtables kept in memory before writes stall waiting for a flush. |
| `shannonstore.api.rocksdb.target.file.size.base` | `67108864` (64 MiB) | Target size for L1 SST files (`target_file_size_base`). |
| `shannonstore.api.rocksdb.block.size` | `16384` (16 KiB) | RocksDB data block (table block) size. |
| `shannonstore.api.rocksdb.block.cache.size` | `33554432` (32 MiB) | RocksDB block cache (LRU). Bump to 128+ MiB only on hosts with memory headroom. |
| `shannonstore.api.rocksdb.bloom.filter.bits` | `10` | Bits-per-key for the metadata-store bloom filter (~10 bits ≈ 1% false positives). |

---

## 3. Data Node Configuration

Data node only.

| Property | Default | Description |
| --- | --- | --- |
| `shannonstore.data.storage.dirs` | `${shannonstore.base.data.dir}/s3-data` | Comma-separated local directories where the Data node stores object chunk/shard files. Multiple dirs on the same filesystem are counted once for capacity — use one directory per physical disk for capacity and parallel I/O. |
| `shannonstore.data.thread.pool.compute.size` | `512` | Size of the Data node's compute/I/O thread pool (chunk read/write, EC encode/decode, encryption, compression). |
| `shannonstore.data.kms.rocksdb.path` | `${shannonstore.base.data.dir}/shannon-data-kms-rocksdb` | Local KMS keystore RocksDB cache. The Data node pulls KEKs from the leader on startup and caches them here. |

---

## 4. Logging & Monitoring

| Property | Default | Description |
| --- | --- | --- |
| `shannonstore.log.mode` | `ASYNC_FILE` | Logging output mode (logback wiring). `ASYNC_FILE` = async logging to rolling files under `log.path`; `RING_BUFFER` = keep recent lines in memory for real-time `/admin/logs/tail`. |
| `shannonstore.log.path` | `${shannonstore.base.data.dir}/logs` | Base directory for rolling log files. |
| `shannonstore.api.prometheus.enabled` | `true` | Whether the API node exposes the Prometheus text-format `/metrics` endpoint. |
| `shannonstore.api.metrics.rocksdb.path` | `${shannonstore.base.data.dir}/shannon-metrics-rocksdb` | RocksDB store for collected time-series monitoring metrics. |
| `shannonstore.api.metrics.retention.days` | `30` | Retention window (days) for stored metrics before the cleanup job purges them. |
| `shannonstore.api.history.rocksdb.path` | `${shannonstore.base.data.dir}/shannon-history-rocksdb` | RocksDB store for operational action/audit history (admin ops, repairs, rebalances, backups). |
| `shannonstore.api.history.retention.days` | `365` | Retention window (days) for action-history entries. |
| `shannonstore.api.metrics.collection.interval.seconds` | `60` | Period (seconds) between collections of system/performance metrics. |

---

## 5. Disk Repair Configuration

| Property | Default | Description |
| --- | --- | --- |
| `shannonstore.api.repair.enabled` | `false` | Enable the automatic disk repair service. When on, each API server periodically scans for shards on dead disks and reconstructs them via EC parity. Manual trigger is always available in the Admin UI. |
| `shannonstore.api.repair.interval.seconds` | `300` | Period (seconds) between disk-repair scan cycles when enabled. |
| `shannonstore.api.repair.grace.period.seconds` | `600` | Grace period (seconds) a disk may be missing before it is declared dead and its shards become eligible for reconstruction. Should exceed normal restart time. |
| `shannonstore.api.repair.max.concurrent` | `4` | Max shard-reconstruction operations running concurrently during repair. |
| `shannonstore.api.repair.min.available.bytes` | `104857600` (100 MiB) | Minimum free space a candidate disk must have to be chosen as a reconstruction target. |

---

## 5b. Bitrot Scrubber Configuration

Periodically reads every chunk on disk and verifies its CRC32C against the checksum recorded at write time; mismatched shards are rebuilt from EC parity. The runtime on/off state is controlled from the Admin UI and persisted in `scrubber.state.file`; the `enabled` key only seeds the initial value when no state file exists.

| Property | Default | Description |
| --- | --- | --- |
| `shannonstore.api.scrubber.enabled` | `false` | Initial enabled state on first boot when no state file is present. Once toggled in the UI, that decision wins on later restarts. |
| `shannonstore.api.scrubber.interval.seconds` | `86400` (1 day) | Interval (seconds) between automatic scrubber cycles when enabled. Lower values increase I/O pressure on Data nodes. |
| `shannonstore.api.scrubber.rate.limit.chunks.per.second` | `10` | Max chunks the scrubber fetches+verifies per second per API node. |
| `shannonstore.api.scrubber.max.concurrent` | `2` | Worker threads the scrubber uses for verification and admin-triggered scans. |
| `shannonstore.api.scrubber.state.file` | `${shannonstore.base.data.dir}/scrubber-state.json` | File where the runtime enabled/disabled toggle is persisted across restarts. The toggle is currently per-node, not cluster-wide. |

---

## 5f. Object Event Notifications

Per-node tuning of the [event-notification dispatcher](event-notifications.md). The
targets and rules themselves are cluster-global and stored in the replicated bucket
config (Admin UI → Event Notifications), **not** here.

| Property | Default | Description |
| --- | --- | --- |
| `shannonstore.api.notification.queue.size` | `10000` | Bounded in-memory dispatch queue per API node. When full, events are dropped (counted) rather than blocking the S3 request path. |
| `shannonstore.api.notification.worker.threads` | `4` | Worker threads draining the queue and delivering to webhook/Kafka targets. |
| `shannonstore.api.notification.max.retries` | `3` | Max delivery retries per event before it is counted as failed (`0` = no retry). |
| `shannonstore.api.notification.retry.backoff.ms` | `200` | Base backoff (ms) between delivery retries; exponential, capped at 5s. |
| `shannonstore.api.notification.config.poll.seconds` | `5` | How often (seconds) each node re-reads the replicated notification config so target/rule/enable changes take effect. |
| `shannonstore.api.bucket.config.rocksdb.path` | `${shannonstore.base.data.dir}/s3-metadata/bucket-config-rocksdb` | Per-node RocksDB that durably holds ALL cluster-global bucket config (policy/CORS/lifecycle/tagging/object-lock/versioning/storage-classes/site-replication/notification) so it survives a full cluster restart. |

---

## 5g. Chunk Rebalance Worker

| Property | Default | Description |
| --- | --- | --- |
| `shannonstore.api.rebalance.enabled` | `false` | Enabled state on FIRST boot only; the Admin UI toggle wins afterwards. |
| `shannonstore.api.rebalance.state.file` | `${shannonstore.base.data.dir}/rebalance-state.json` | Where the runtime rebalance toggle is persisted (survives restart), one file per node. |

---

## 6. Operational Tuning

| Property | Default | Description |
| --- | --- | --- |
| `shannonstore.api.message.handler.thread.pool.size` | `48` | Threads processing inbound internal NIO messages (chunk/metadata/KMS RPCs from peers). In-code default `max(16, cores × 4)`. |
| `shannonstore.api.admin.handler.thread.pool.size` | `16` | Threads running admin-triggered async tasks (repair/rebalance scans, backup/restore coordination). |
| `shannonstore.api.admin.proxy.connect.timeout.ms` | `5000` | TCP connect timeout (ms) when an API node proxies an admin write to the leader. |
| `shannonstore.api.admin.proxy.read.timeout.ms` | `30000` | Read/response timeout (ms) for proxied admin-to-leader forwarding. |
| `shannonstore.api.admin.logs.proxy.read.timeout.ms` | `10000` | Read timeout (ms) for proxying the `/admin/logs/tail` stream to a peer (kept low because log tail is interactive). |
| `shannonstore.api.admin.logs.default.lines` | `100` | Default number of trailing log lines returned by `/admin/logs/tail` when `?lines=` is omitted. |
| `shannonstore.api.admin.cors.max.age.seconds` | `86400` (24 h) | Value sent in the CORS `Access-Control-Max-Age` header for admin API preflight requests. |
| `shannonstore.api.s3.list.objects.max.keys` | `1000` | Max object keys returned in a single ListObjects(V2) page; clients paginate via continuation tokens. |
| `shannonstore.network.nio.write.timeout.ms` | `30000` | Per-connection write timeout (ms) for the internal NIO layer; a queued outbound message that can't flush within the window fails. |
| `shannonstore.network.nio.connection.read.buffer.size` | `1048576` (1 MiB) | Per-connection heap read buffer for the internal NIO server, allocated per accepted connection. |
| `shannonstore.network.rpc.default.timeout.seconds` | `30` | Default timeout (seconds) for a synchronous NIO `sendAndReceive` RPC when the caller specifies none. |
| `shannonstore.network.rpc.connect.timeout.ms` | `5000` | TCP connect timeout (ms) for `NioRpcClient.connect()` when establishing an inter-node channel. |
| `shannonstore.network.rpc.connect.poll.sleep.ms` | `100` | Poll/sleep granularity (ms) of the busy-wait loop while `NioRpcClient.connect()` waits for the channel. |
| `shannonstore.network.http.header.buffer.size` | `65536` (64 KiB) | Max buffer (bytes) for parsing an incoming HTTP request line + headers; larger header blocks are rejected. |
| `shannonstore.network.http.body.read.buffer.size` | `8192` (8 KiB) | Heap buffer (bytes) per read step when draining an HTTP request body. |
| `shannonstore.network.http.response.stream.chunk.size` | `8388608` (8 MiB) | HTTP response streaming chunk size (bytes) for large S3 GETs. |
| `shannonstore.api.iam.policy.pattern.cache.size` | `10000` | Max entries in the IAM `PolicyEvaluator` regex-pattern cache; cleared and rebuilt when full. |
| `shannonstore.api.sts.default.session.duration.seconds` | `3600` | Default STS session duration (seconds) when clients omit `DurationSeconds`. AWS-compatible range is [900, 43200]. |
| `shannonstore.api.metadata.rpc.read.timeout.seconds` | `5` | Timeout (seconds) for a metadata GET RPC sent to the HRW-owning API node when the local node is not the owner. |
| `shannonstore.api.metadata.rpc.write.timeout.seconds` | `10` | Timeout (seconds) for a metadata WRITE/DELETE RPC forwarded to the HRW-owning primary. Longer than the read timeout because SYNC writes wait on replica ACKs. |
| `shannonstore.api.metadata.replication.mode` | `PULL` | Metadata replication mode. `PULL` = primary acks immediately, replicas catch up (lowest latency, eventually consistent). `ASYNC_PUSH` = primary fans out off the hot path. `SYNC` = primary waits for every replica ACK (strong consistency, highest tail latency). |
| `shannonstore.api.metadata.replication.factor` | `2` | HRW replication factor *R* for metadata. Each key is owned by the top-*R* API nodes by HRW score. `R=1` = no replication; `R=2` is the safe default. |
| `shannonstore.api.metadata.rocksdb.shards` | `16` | Number of RocksDB sub-stores backing the metadata index; 65,536 vbuckets are split evenly across them. Must evenly divide 65,536. **Changing this in a live deployment requires data migration.** |
| `shannonstore.api.metadata.rocksdb.dirs` | _(empty)_ | *Optional.* Comma-separated disk paths the metadata shards are striped across. Empty → use the single `index.rocksdb.path`. Multiple paths → shard *i* lives at `dirs[i % len]/shard-i/`. |
| `shannonstore.api.metadata.reconcile.enabled` | `false` | Enable the reconciliation sweep that deletes local metadata copies left on a node no longer an HRW owner (akin to `nodetool cleanup`). Opt in once the cluster is stable. |
| `shannonstore.api.metadata.reconcile.interval.seconds` | `300` | Reconciliation sweep period (seconds). Minimum 30s. |
| `shannonstore.api.metadata.reconcile.warmup.seconds` | `60` | Wait after cluster-ready before the first sweep, giving bootstrap snapshots time to land. |
| `shannonstore.api.metadata.reconcile.max.deletions.per.cycle` | `1000` | Hard cap on entries deleted per cycle, bounding blast radius if HRW returns a wrong answer. |
| `shannonstore.api.metadata.reconcile.dry.run` | `false` | When true, the sweep logs what it would delete but deletes nothing — useful for verifying ownership math first. |
| `shannonstore.api.part.buffer.peer.fetch.timeout.seconds` | `10` | Timeout (seconds) when an API node fetches a not-yet-flushed multipart part from the peer that buffered it. |
| `shannonstore.api.iam.sync.interval.seconds` | `60` | Period (seconds) of the background IAM sync loop pulling the latest IAM bundle from the leader. |
| `shannonstore.api.iam.sync.min.interval.ms` | `30000` | Minimum spacing (ms) between successive IAM syncs (rate-limit guard). |
| `shannonstore.api.iam.reload.retry.sleep.ms` | `500` | Sleep (ms) between retries when a follower reloads the IAM bundle from the cluster. |
| `shannonstore.api.kms.restore.retry.sleep.ms` | `500` | Sleep (ms) between retries when restoring the KMS keystore from the cluster at startup. |
| `shannonstore.api.s3.metadata.poll.sleep.ms` | `300` | Poll sleep (ms) while a large-object streaming read waits for the object's metadata to become available. |
| `shannonstore.api.s3.decoded.part.cache.bytes` | `33554432` (32 MiB) | Total-bytes ceiling for the decoded-part LRU cache (`BoundedBytesLruCache`). Bounded by bytes, not count, since a single decoded segment can be tens of MB. |
| `shannonstore.api.cluster.node.discovery.retry.sleep.ms` | `1000` | Sleep (ms) between node-discovery retries while an API node waits for cluster membership at startup. |
| `shannonstore.api.cluster.node.discovery.max.retries` | `5` | Max node-discovery retry attempts before proceeding with rebalance bootstrap. |
| `shannonstore.kms.key.sync.retry.count` | `3` | Retry attempts a replica makes when syncing a missing KMS key from the source node. |
| `shannonstore.kms.key.sync.retry.sleep.ms` | `2000` | Sleep (ms) between KMS key-sync retry attempts on a replica. |
| `shannonstore.api.metrics.retention.cleanup.interval.hours` | `12` | How often (hours) the metrics retention cleanup job runs to purge samples older than `metrics.retention.days`. |
| `shannonstore.log.ring.buffer.capacity` | `1000` | Capacity (recent log lines) of the in-memory ring buffer feeding `/admin/logs/tail`. Only material when `log.mode=RING_BUFFER`. |
| `shannonstore.data.startup.leader.wait.sleep.ms` | `1000` | Data node: sleep (ms) between polls while waiting for an API leader at startup. **Caveat:** in-code default is 500; the file sets 1000. |
| `shannonstore.data.disk.info.refresh.interval.seconds` | `60` | Data node: period (seconds) for recomputing per-disk `DiskInfo` and refreshing the payload advertised in ZooKeeper. |

---

## S3 Backup

Most operators set the runtime backup values (enabled, endpoint, bucket, keys) from the Admin UI; those are persisted on the leader and override the keys below, which only control the backup loop's internals.

| Property | Default | Description |
| --- | --- | --- |
| `shannonstore.backup.tick.interval.seconds` | `60` | How often (seconds) the backup ticker wakes to evaluate (interval, leader, enabled). |
| `shannonstore.backup.history.cap` | `200` | Cap on the in-memory recent-history list shown in the Admin UI. The S3-side history index is independent and trimmed by retention days. |
| `shannonstore.backup.rpc.timeout.ms` | `1800000` (30 min) | Per-data-node fan-out timeout (ms) for a backup run request; long because a full snapshot of a chunk store can take a while. |

---

## 7. Bootstrap / Restore Loops

Knobs for the startup-time loops that pull IAM/KMS state from the leader and poll for cluster readiness.

| Property | Default | Description |
| --- | --- | --- |
| `shannonstore.api.iam.reload.max.attempts` | `60` | Max attempts a follower API node makes to pull the IAM bundle from the leader before continuing with local state. |
| `shannonstore.api.kms.restore.max.attempts` | `60` | Max attempts a follower API node makes to pull the KMS keystore from the leader before continuing. Pair with `kms.restore.retry.sleep.ms`. |
| `shannonstore.api.cluster.ready.poll.interval.ms` | `500` | How often (ms) the cached cluster-ready ZK flag is refreshed. |
| `shannonstore.api.cluster.leader.poll.sleep.ms` | `200` | Sleep (ms) between bootstrap leader-detection polls during `ClusterManager` startup (capped by `cluster.leader.election.timeout.ms`). |
| `shannonstore.data.startup.leader.ready.poll.sleep.ms` | `500` | Data node: sleep (ms) between polls of the cluster-ready/leader ZK flag before pulling the KMS keystore. Floored at 50 ms in code. |
| `shannonstore.data.kms.pull.max.attempts` | `60` | Data node: max attempts to fetch the KMS keystore from the leader at startup. Pair with `data.startup.leader.wait.sleep.ms`. |
| `shannonstore.data.kms.pull.rpc.timeout.seconds` | `5` | Data node: per-attempt RPC timeout (seconds) for the startup KMS pull. |

---

## Hot-reload vs restart

Most keys take effect at process start. A small subset is mutable at runtime through the Admin UI / Admin REST, and persisted so the change survives restart:

| Key area | Mutable at runtime? | How |
| --- | --- | --- |
| Maintenance mode | yes | Admin UI / admin REST |
| Bitrot scrubber enable / pause | yes | Admin UI toggle (per-node, persisted to `scrubber.state.file`) |
| Disk-repair manual trigger | yes | Admin UI |
| S3 backup schedule / target | yes | Admin UI (persisted on the leader, overrides the file keys) |
| EC scheme, ports, ZK topology, KMS master key | no | restart required |

A live cluster's storage layout and addressing should not change without an explicit deploy; everything that's a *policy* (when to scrub, when to back up) responds to operator judgement without a restart.

## See also

- [Download & Install](../installation/installation.md) — the bootstrap path that consumes these keys.
- [Getting Started](../installation/getting-started.md) — first bucket, key, and object.
- [Performance Optimization](performance.md) — the runtime knobs that shape latency and throughput.
- [Encryption & Key Management](kms.md) — the master-key requirement and rotation flow.
- [Monitoring & Metrics](monitoring.md) — the metrics surfaces produced by the keys in the metrics section.
