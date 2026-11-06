# Network & Internal Communication

Three traffic planes leave a ShannonStore node, each tuned for a different shape of work. Operators should understand the split to size sockets, plan firewalls, and reason about latency.

```text
   ┌───────────────────────────────────┐
   │  S3 client traffic   :8080        │  AWS SDKs, applications, range
   │  HTTP/1.1 keep-alive + HTTP/2 h2c │  reads, multipart uploads
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

The S3 plane (8080) is served by ShannonStore's own `java.nio` HTTP server (`NioHttpServer`), which speaks HTTP/1.1 with persistent connections and HTTP/2 over clear text (h2c) on the same port. The admin plane (8888) is a Netty HTTP/1.1 server. The internal plane (9090) is a custom NIO framed protocol, kept narrow so the hot path between API nodes and data nodes never pays HTTP parsing overhead.

## S3 client plane (port 8080)

The outward face of the cluster. AWS wire format over **HTTP/1.1 with keep-alive** or **HTTP/2 (h2c)**. See [S3-Compatible API](s3-compatible-api.md) for the operations and [HTTP Protocols & Client Tuning](http-protocols.md) for connection handling, HTTP/2, and client pool settings.

How it is served:

- **One selector thread** accepts connections, reads request headers without blocking, and holds idle keep-alive connections. An idle connection costs a socket and a small header buffer, never a thread.
- **A worker pool** (`cores × netty.worker.threads.multiplier`) runs each request once its headers are complete. Workers also run each HTTP/2 stream.
- After the response, the connection returns to the selector for the client's next request. It is closed instead when the response could not be cleanly delimited or the client asked to close.

Tuning levers worth knowing:

| Key | Default | Effect |
| --- | --- | --- |
| `shannonstore.api.s3.port` | `8080` | Bind port for client traffic. |
| `shannonstore.api.netty.worker.threads.multiplier` | `4` | Request worker pool size = `multiplier × availableProcessors()`. The name is historical: the S3 port does not use Netty. |
| `shannonstore.network.http.keepalive.enabled` | `true` | HTTP/1.1 persistent connections. |
| `shannonstore.network.http.keepalive.idle.timeout.ms` | `75000` | Idle or slow-header connections are closed after this. Keep it longer than the proxy's and the SDK's idle timeouts. |
| `shannonstore.network.http.max.connections` | `0` (unlimited) | Cap on open client connections. |
| `shannonstore.network.http.accept.backlog` | `1024` | Listen backlog (the JDK default is 50). |
| `shannonstore.network.http2.enabled` | `true` | HTTP/2 h2c on the same port. See [HTTP Protocols](http-protocols.md#configuration) for the HTTP/2 limits. |
| `shannonstore.api.s3.fetch.timeout.seconds` | `120` (shipped; code default 60) | Per-shard read deadline propagated to the internal plane. |
| `shannonstore.api.s3.chunked.stream.size` | `1 MiB` | Write-chunk size for streamed GET bodies. Responses carry `Content-Length`; this is not `Transfer-Encoding: chunked`. |
| `shannonstore.api.s3.circular.buffer.size` | `128 MiB` (shipped; code default 64 MiB) | Per-request ring buffer for the streaming download path. |
| `shannonstore.api.s3.small.object.threshold` | `64 MiB` | Cutoff between in-memory PUT/GET and streaming. Objects below this short-circuit through a single buffer; above, the streaming path engages. |

TLS termination happens at the upstream proxy. See [Nginx Reverse Proxy](../operations/nginx-proxy.md) for the recommended configuration that fronts both 8080 and 8888. nginx forwards to the S3 port over pooled HTTP/1.1 keep-alive connections. To carry HTTP/2 all the way to the API nodes, use a proxy that speaks HTTP/2 to upstreams (HAProxy `proto h2`, Envoy). See [HTTP Protocols](http-protocols.md#where-http2-helps-and-the-nginx-caveat).

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

Distinct pools serve the planes so a runaway worker on one path never starves another:

| Pool | Default size | Workload |
| --- | --- | --- |
| S3 request workers (`NioHttpServer`) | `netty.worker.threads.multiplier × cpus` (default `4 × cpus`) | One HTTP/1.1 request or HTTP/2 stream each on 8080. Accept, header reads, and idle connections stay on one selector thread. |
| Admin Netty event loops | 1 boss + `max(2, cpus)` workers | Admin REST/UI on 8888 |
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

The internal plane runs plaintext by design — both ends are inside the cluster's private network and the messages are already protected by the cluster's at-rest encryption pipeline (shards leave the API node already ciphertext). Messages that carry credentials are KMS-encrypted before they go on the wire: the IAM / bucket-config snapshot, a follower's bucket-config change forwarded to the leader (it can hold replication and site-replication secret keys and webhook credentials), the backup and restore requests the leader sends to every data node (they hold the backup bucket's secret key), and admin-socket requests forwarded to the leader. Adding TLS to the internal plane would double-encrypt payloads that are already secure. Wrap the cluster's network at the perimeter (VPC, WireGuard, etc.) rather than wrapping individual peer links.

Admin requests that only the leader may serve (IAM, storage classes, backup settings) and that land on a follower are relayed to the leader's admin port as they came in, so they are as protected as the operator's own connection: run the admin plane over TLS.

The S3 and Admin planes are HTTP and *should* run over TLS in production. Terminate at the upstream nginx; see [Nginx Reverse Proxy](../operations/nginx-proxy.md).

## Diagnostics

Each plane logs distinct slow-path lines so operators can pinpoint contention:

| Log signature | Plane | Likely cause |
| --- | --- | --- |
| `Timeout (Ns) waiting for connection permit to <host:port>` | internal NIO | `connection.pool.size.per.host` too small for the current fan-out, or the peer is unreachable |
| `S3 port 8080: N connections open (max M); refusing new connections` | S3 (8080) | `network.http.max.connections` reached. Raise it, or find the client that is not reusing connections. |
| `Headers too large, closing connection` | S3 (8080) | request headers exceed `network.http.header.buffer.size` (64 KiB) |
| `Streamed X bytes but announced Content-Length Y; closing the connection` | S3 (8080) | a streamed GET body ended early (a backend read failed mid-stream). The client sees a short read and retries. |
| `NioHttpServer started on port … (keep-alive on, idle timeout …, HTTP/2 h2c on …)` (startup) | S3 (8080) | confirms the effective connection settings |

Connections that sit idle past `network.http.keepalive.idle.timeout.ms` are closed silently. That is normal and is not an error.

Prometheus (`/metrics` on the admin port when enabled) exposes S3 request latency per operation as the Micrometer timer `s3_api_requests_seconds{method="…"}` (`_count`, `_sum`, `_max`), plus the JVM, GC, thread, and CPU binders.

## See also

- [Configuration Management](configuration.md) — the resolution chain that produces every knob mentioned here.
- [Performance Optimization](performance.md) — how the pools and buffers interact under load.
- [Nginx Reverse Proxy](../operations/nginx-proxy.md) — the recommended TLS termination + routing setup.
- [S3-Compatible API](s3-compatible-api.md) — what the 8080 surface speaks.
- [HTTP Protocols & Client Tuning](http-protocols.md) — keep-alive, HTTP/2 (h2c), client pool settings.
- [Authentication & Authorization](auth-authz.md) — what the 8888 surface requires.
