# Network & Internal Communication

Three traffic planes leave a ShannonStore node, each tuned for a different shape of work. Operators should understand the split to size sockets, plan firewalls, and reason about latency.

```text
   ┌───────────────────────────────────┐
   │  S3 client traffic   :8080        │  AWS SDKs, applications, range
   │   Netty HTTP/1.1, AWS wire format │  reads, multipart uploads
   └────────────────┬──────────────────┘
                    │  (TCP, optional TLS at the proxy)
                    ▼
   ┌───────────────────────────────────────────────────────────┐
   │                  ShannonStore API node                    │
   └────────────────┬─────────────────────────┬───────────────┘
                    │                         │
   Admin REST :8888 │                         │  Internal NIO :9090
   ──────────────── │  Bearer JWT, Admin UI   │  ─────────────────
   small messages, │                         │  shard / metadata
   high RPS at boot │                         │  payloads — biggest
   then steady     │                         │  fan-out of the cluster
                    │                         │
                    ▼                         ▼
              Admin tooling             Peer API + Data nodes
              (operators,                (NIO framed protocol,
              cross-product              pooled connections,
              sweepers)                  optional compression)
```

The HTTP planes (8080, 8888) are handled by Netty with the standard HTTP/1.1 pipeline. The internal plane (9090) is a custom NIO framed protocol — kept narrow so the hot path between API nodes and data nodes never pays HTTP parsing overhead.

## S3 client plane (port 8080)

The outward face of the cluster. Standard HTTP/1.1, AWS wire format, see [S3-Compatible API](s3-compatible-api.md) for the protocol shape. Tuning levers worth knowing:

| Key | Default | Effect |
| --- | --- | --- |
| `shannonstore.api.s3.port` | `8080` | Bind port for client traffic. |
| `shannonstore.api.netty.worker.threads.multiplier` | `4` | Netty worker pool size = `multiplier × availableProcessors()`. |
| `shannonstore.api.s3.fetch.timeout.seconds` | `60` | Per-shard read deadline propagated to the internal plane. |
| `shannonstore.api.s3.chunked.stream.size` | `1 MiB` | Chunk size advertised in `Transfer-Encoding: chunked` for streamed GETs. |
| `shannonstore.api.s3.circular.buffer.size` | `64 MiB` | Per-request ring buffer for the streaming download path. |
| `shannonstore.api.s3.small.object.threshold` | `64 MiB` | Cutoff between in-memory PUT/GET and streaming. Objects below this short-circuit through a single buffer; above, the streaming path engages. |

TLS termination happens at the upstream proxy (nginx). See [Nginx Reverse Proxy](../operations/nginx-proxy.md) for the recommended configuration that fronts both 8080 and 8888.

## Admin plane (port 8888)

JWT-authenticated REST. Used by:

- The Admin UI bundled with the release.
- Cross-product tooling (chango, ontul, kiok, neorunbase) that drives IAM provisioning, access-key minting, and cluster-status checks.
- Backup / restore scripts.
- Maintenance-mode toggles, scrubber controls, disk-repair triggers.

This plane is **never** intended to face the public internet — bind it to a management VLAN or expose it through an authenticated reverse proxy that further restricts source IP ranges. The default-credential gate (`requirePasswordChange = true` on the bootstrap admin) is the security back-stop, not the front line.

## Internal NIO plane (port 9090)

Where the bulk of the cluster's bytes move. Three workloads share this transport:

| Workload | Sender | Receiver |
| --- | --- | --- |
| Shard read / write | API node | data node |
| Metadata replication | API leader | API follower |
| Membership / heartbeat | every node | every node |
| Maintenance-mode broadcast | API leader | API followers |
| KMS sync | API leader | data nodes |

The framing is fixed-header + variable-payload binary. No HTTP parsing, no JSON: each message is a 4-byte length + opcode + payload. Payloads are protobuf-like compact records (the project's `Message` class) so the wire size is minimal even before any compression.

### Connection pooling

`ConnectionPool` retains idle NIO connections per peer (`host:port`) so the hot read/write path doesn't pay TCP-handshake cost per request.

```text
   ConnectionPool
     ├── idlePools["data-1:9090"]  → [client0, client1, …, clientN]
     ├── idlePools["data-2:9090"]  → [client0, …]
     └── idlePools["api-2:9090"]   → [client0, …]
```

Knobs:

| Key | Default | Effect |
| --- | --- | --- |
| `shannonstore.api.connection.pool.size.per.host` | `64` | Cap on idle clients retained per peer. |
| `shannonstore.api.connection.pool.acquire.timeout.seconds` | `30` | Blocking acquire deadline. |
| `shannonstore.api.connection.pool.warmup.enabled` | `true` | Pre-open connections at startup so the first request after a restart doesn't pay TCP handshake. |

Direct buffers (off-heap) are used for inbound and outbound payloads — the JVM heap stays out of the IO path entirely on platforms where `sun.nio.ch.DirectBuffer` is available. Operators can disable direct buffers via the same constructor flag if a constrained environment needs heap-only IO.

Failed sockets are evicted from the pool via `invalidateConnection()` so a transient peer crash doesn't poison the pool. The next acquire opens a fresh connection and (eventually) succeeds when the peer recovers.

## Compression on the internal plane

Inter-node payloads compress transparently above a configurable size threshold:

| Key | Default | Effect |
| --- | --- | --- |
| `shannonstore.network.compression.type` | `SNAPPY` | Algorithm: only `NONE` / `SNAPPY` are actually supported at this layer — see caveat below. |
| `shannonstore.network.compression.threshold` | `1024` | Bytes — below this the payload ships uncompressed. |

**Caveat:** unlike the object-level `shannonstore.api.s3.object.compression.type` (which does support `NONE`/`SNAPPY`/`GZIP`/`ZSTD` — see [Data Compression](data-compression.md)), `NetworkHelper.compress(byte[], String type)` on the internal NIO plane ignores its `type` argument entirely and always compresses with Snappy; `decompress()` only recognizes the Snappy header byte and throws for anything else. Setting `shannonstore.network.compression.type` to `GZIP` or `ZSTD` has no effect — Snappy is used regardless. The shipped `shannonstore.properties` template itself documents this: "Supported: NONE, SNAPPY" for this key (as opposed to the four-way list on the object-level key two sections later).

The 1 KiB default is chosen so that small control messages (heartbeats, membership pings) don't pay the compression latency, while shard payloads (the dominant byte flow) do. The compression header is a single byte at the start of the payload, so a peer running a different setting still decodes correctly (Snappy either way) — change the setting on a rolling restart without coordination.

## Thread-pool isolation

Three distinct pools serve the three planes so a runaway worker on one path never starves another:

| Pool | Default size | Workload |
| --- | --- | --- |
| Netty boss + worker | `4 × cpus` | Accept + read/write on 8080 and 8888 |
| Fetch pool | `max(16, 4 × cpus)` | Per-request shard fan-out reads |
| Chunk pool | `max(16, 4 × cpus)` | EC encode / decode, KMS unwrap, compression |

The fetch and chunk pools are independently sized via `shannonstore.api.fetch.thread.pool.size` and `shannonstore.api.chunk.thread.pool.size`. The split exists because shard reads are network-bound (waiting for sockets) while chunk work is CPU-bound (RS decoding, AES-GCM, snappy/zstd). Doubling one without the other has no global effect.

## Firewall and routing

For a cluster split across racks or AZs:

| Direction | Port | Required? |
| --- | --- | --- |
| Client → API node | 8080 | yes |
| Operator → API node | 8888 | yes (management VLAN) |
| API node → API node | 9090 | yes |
| API node → data node | 9090 | yes |
| Data node → API node | 9090 (callback) | yes |
| Data node → data node | — | not used |

Data nodes do not talk directly to each other. Every cluster-wide message either passes through the API tier or is broadcast from the leader, so the data-tier subnet only needs ingress from the API tier.

## TLS

The internal plane runs plaintext by design — both ends are inside the cluster's private network and the messages are already protected by the cluster's at-rest encryption pipeline (shards leave the API node already ciphertext). Adding TLS to the internal plane would double-encrypt payloads that are already secure. Wrap the cluster's network at the perimeter (VPC, WireGuard, etc.) rather than wrapping individual peer links.

The S3 and Admin planes are HTTP and *should* run over TLS in production. Terminate at the upstream nginx; see [Nginx Reverse Proxy](../operations/nginx-proxy.md).

## Diagnostics

Each plane logs distinct slow-path lines so operators can pinpoint contention:

| Log signature | Plane | Likely cause |
| --- | --- | --- |
| `Acquire timeout on connection pool …` | internal NIO | `connection.pool.size.per.host` too small for current fan-out |
| `Fetch timeout for chunk …` | internal NIO | data node slow, raise `s3.fetch.timeout.seconds` or investigate the peer |
| `Netty channel idle timeout` | S3 (8080) | client opened a TCP connection and idled past the keep-alive window |
| `Compression skipped — payload below threshold` (debug) | internal NIO | informational; tune `network.compression.threshold` to be more aggressive |

Prometheus exposes per-plane counters (`shannonstore_http_requests_total{port="8080"}`, etc.) so a grafana panel can visualize the distribution at a glance.

## See also

- [Configuration Management](configuration.md) — the resolution chain that produces every knob mentioned here.
- [Performance Optimization](performance.md) — how the pools and buffers interact under load.
- [Nginx Reverse Proxy](../operations/nginx-proxy.md) — the recommended TLS termination + routing setup.
- [S3-Compatible API](s3-compatible-api.md) — what the 8080 surface speaks.
- [Authentication & Authorization](auth-authz.md) — what the 8888 surface requires.
