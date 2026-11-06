# HTTP Protocols & Client Tuning

The S3 port (`8080` by default) speaks **HTTP/1.1 with persistent connections** and
**HTTP/2 over clear text (h2c)** on the same socket. Both are served by ShannonStore's own
`java.nio` HTTP server (`NioHttpServer`) — not Netty, and no third-party HTTP/2 library. The
admin port (`8888`) is a separate Netty server and the internal node plane (`9090`) a custom
binary protocol; neither is affected by anything on this page.

Why it matters: an S3 workload is many small requests. On a lakehouse table, a query planner
reads manifests, footers and column chunks as hundreds or thousands of GETs, each typically
under a few MiB. When every request opens a new TCP connection — and, behind a TLS proxy, a
new TLS session — the handshakes dominate the latency, the client's connection pool churns,
and sockets pile up in `TIME_WAIT`. HTTP/1.1 keep-alive removes the per-request handshake;
HTTP/2 additionally multiplexes many concurrent requests over one connection.

```text
   client ──HTTP/1.1 keep-alive──▶ ┌─────────────────────────────┐
   client ──h2c (prior knowledge)─▶ │  :8080  NioHttpServer       │
   client ──HTTP/1.1 Upgrade: h2c─▶ │   same port, same S3 logic   │
                                   └─────────────────────────────┘
   nginx  ──HTTP/1.1 keep-alive (upstream keepalive pool)──▶ :8080
   HAProxy / Envoy ──HTTP/2 (proto h2 / http2 upstream)────▶ :8080
```

## HTTP/1.1 persistent connections

A connection stays open after a response and carries the next request. Clients that pool
connections (AWS SDKs, s3a, boto3/urllib3, aws-cli, pyarrow) and proxies with an upstream
keep-alive pool (nginx `keepalive`) reuse it instead of reconnecting.

The server keeps a connection only when the response is unambiguously delimited, and closes it
otherwise — so a mis-framed response can never be read as the start of the next one:

| Situation | Connection after the response |
| --- | --- |
| Normal response with `Content-Length` (or a bodyless status: 1xx/204/304, or a HEAD) | **kept** (`Connection: keep-alive`) |
| Client sent `Connection: close`, or HTTP/1.0 without `Connection: keep-alive` | closed |
| Handler answered before reading the whole request body (e.g. `403` to a `PUT`) | kept if the unread rest is ≤ `keepalive.max.drain.bytes` (it is read and discarded), closed otherwise |
| Request carried both `Content-Length` and `Transfer-Encoding: chunked`, or an invalid `Content-Length` | closed (request-smuggling shape) |
| Internal error, no response, or a streamed body shorter than its `Content-Length` | closed |

Other properties of the implementation:

- **Pipelining.** Requests sent back-to-back without waiting are answered in order.
- **Idle connections cost no thread.** An idle keep-alive connection lives on the selector thread with a small header buffer (4 KiB, grown on demand up to 64 KiB). Request threads (`cores × shannonstore.api.netty.worker.threads.multiplier`) are only taken while a request is being served, so thousands of pooled client connections do not need thousands of threads.
- **Idle timeout.** A connection idle for `keepalive.idle.timeout.ms` (75 s), or that has not finished sending its request headers in that time, is closed. This also bounds slow-header ("slowloris") connections.
- **HEAD** responses announce `Content-Length` but never send body bytes, including error responses.
- **`Expect: 100-continue`** is answered immediately with `100 Continue`, so SDK uploads don't stall for the client's continue timeout.

!!! note "Before 1.0.0 builds of 2026-11"
    Earlier builds closed the connection after every response (`Connection: close`), so
    client and nginx connection pools reused nothing and every S3 call paid a TCP handshake.
    No configuration change is needed to benefit; the documented nginx `upstream keepalive`
    pool now actually takes effect.

## HTTP/2 (h2c)

HTTP/2 multiplexes many concurrent requests ("streams") over one TCP connection, compresses
headers (HPACK) and applies per-stream flow control. ShannonStore serves it in clear text on
the S3 port; TLS (and ALPN negotiation of `h2`) is the job of the proxy in front, exactly as for
HTTP/1.1.

A connection becomes HTTP/2 in one of two ways:

| Mode | How | Who uses it |
| --- | --- | --- |
| **Prior knowledge** | the connection starts with the HTTP/2 preface | HAProxy `proto h2`, Envoy http2 upstream, `curl --http2-prior-knowledge`, gRPC-style clients |
| **Upgrade** | an HTTP/1.1 request with `Upgrade: h2c` + `HTTP2-Settings` is answered `101 Switching Protocols`, then that request continues as stream 1 | JDK `HttpClient` on `http://` URLs, `curl --http2` on `http://` |

Only a request **without a body** is upgraded; a request with a body that asks to upgrade is
served as HTTP/1.1 (RFC 9113 allows the server to ignore the upgrade). HTTP/1.1 clients are
unaffected by HTTP/2 being enabled.

Behaviour worth knowing:

- **Same S3 semantics.** Every S3 operation, signature (SigV4 including presigned URLs), header and error behaves identically over HTTP/2. `:authority` stands in for `Host` when signing is verified.
- **Flow control protects the server.** A client may send at most `initial.window.size` request-body bytes per stream (and `connection.window.size` per connection) before the server returns credit — and credit is returned only as the request is actually processed. A slow backend therefore slows the uploading client instead of growing server memory.
- **Fairness and limits.** Up to `max.concurrent.streams` requests per connection run at once; more are refused with `REFUSED_STREAM`, which clients retry. Oversized request headers get `431`. A stream whose client stops reading (or stops sending its body) for `stream.timeout.ms` is reset, so a stalled client cannot pin a server thread.
- **Errors stay on their stream.** A failing request gets its own `500` (or `RST_STREAM`); the other streams and the connection continue. Protocol violations end the connection with `GOAWAY` and an error code.
- **Idle HTTP/2 connections** with no active streams are closed with `GOAWAY(NO_ERROR)` after `keepalive.idle.timeout.ms`.

### Where HTTP/2 helps — and the nginx caveat

nginx terminates HTTP/2 from clients but **always speaks HTTP/1.1 to upstreams** (`proxy_pass`
has no HTTP/2 mode). So:

| Front end | Client ↔ proxy | Proxy ↔ ShannonStore | Notes |
| --- | --- | --- | --- |
| nginx (documented default) | HTTP/1.1 or HTTP/2 over TLS (`listen 443 ssl http2`) | **HTTP/1.1 keep-alive** | upstream connections are pooled and reused — see [Nginx Reverse Proxy](../operations/nginx-proxy.md) |
| HAProxy | HTTP/1.1 / HTTP/2 over TLS | **HTTP/2** (`server … proto h2`) | many client requests multiplexed over few upstream connections |
| Envoy | HTTP/1.1 / HTTP/2 over TLS | **HTTP/2** (`http2_protocol_options` on the cluster) | |
| Direct (same LAN, no TLS) | h2c or HTTP/1.1 | — | clients that speak h2c |

HAProxy example (TLS from clients, HTTP/2 to the S3 port):

```text
frontend s3_in
    bind :443 ssl crt /etc/haproxy/certs/s3.pem alpn h2,http/1.1
    mode http
    default_backend shannon_s3

backend shannon_s3
    mode http
    balance roundrobin
    timeout server 300s
    timeout http-keep-alive 60s          # below keepalive.idle.timeout.ms (75 s)
    server api1 10.0.0.11:8080 proto h2 check
    server api2 10.0.0.12:8080 proto h2 check
```

## Configuration

All keys live in `shannonstore.properties` (system-property overrides work as usual — see
[Configuration Management](configuration.md)). They are read at startup; changing them needs a
rolling restart of the API nodes.

### HTTP/1.1 and connection handling

| Key | Default | Effect |
| --- | --- | --- |
| `shannonstore.network.http.keepalive.enabled` | `true` | Persistent connections. `false` restores close-after-every-response. |
| `shannonstore.network.http.keepalive.idle.timeout.ms` | `75000` | Close a connection idle (or stuck mid-headers) this long. Keep it **longer** than the idle timeout of whatever pools connections in front: nginx upstream `keepalive_timeout` (60 s), AWS SDK `connectionMaxIdleTime` (60 s), s3a `fs.s3a.connection.idle.time` (60 s) — the client side must close first. |
| `shannonstore.network.http.keepalive.max.drain.bytes` | `1048576` | Unread request body the server will read and discard to keep the connection after an early answer. |
| `shannonstore.network.http.max.connections` | `0` (unlimited) | Cap on open client connections (HTTP/1.1 + HTTP/2). Beyond it new connections are closed immediately and a warning is logged. Check `ulimit -n` too. |
| `shannonstore.network.http.accept.backlog` | `1024` | Listen backlog (the JDK default is 50). Effective value is capped by the OS (`net.core.somaxconn`). |
| `shannonstore.network.http.header.initial.buffer.size` | `4096` | Initial per-connection header buffer, grown to `shannonstore.network.http.header.buffer.size` (64 KiB) only when needed. |
| `shannonstore.api.netty.worker.threads.multiplier` | `4` | Request worker threads = `cores × multiplier` (the name is historical; the S3 port does not use Netty). Each worker serves one HTTP/1.1 request or one HTTP/2 stream at a time. |

### HTTP/2

| Key | Default | Effect |
| --- | --- | --- |
| `shannonstore.network.http2.enabled` | `true` | Serve h2c (prior knowledge) on the S3 port. |
| `shannonstore.network.http2.h2c.upgrade.enabled` | `true` | Honour `Upgrade: h2c`. |
| `shannonstore.network.http2.max.concurrent.streams` | `128` | Concurrent requests per connection (`SETTINGS_MAX_CONCURRENT_STREAMS`). |
| `shannonstore.network.http2.initial.window.size` | `1048576` | Request-body bytes per stream before credit is needed (`SETTINGS_INITIAL_WINDOW_SIZE`). Larger = faster uploads on high-latency links. |
| `shannonstore.network.http2.connection.window.size` | `16777216` | Request-body bytes per connection — the bound on upload data buffered per HTTP/2 connection. |
| `shannonstore.network.http2.max.frame.size` | `16384` | Largest frame payload accepted (16384–16777215). Outgoing frames follow the client's setting. |
| `shannonstore.network.http2.max.header.list.size` | `65536` | Decoded request-header bytes; larger requests get `431`. |
| `shannonstore.network.http2.header.table.size` | `4096` | HPACK dynamic table the client may use for request headers. |
| `shannonstore.network.http2.max.queued.write.bytes` | `4194304` | Response bytes queued per connection before writers wait for the socket. |
| `shannonstore.network.http2.stream.timeout.ms` | `60000` | How long a stream waits on its client (flow-control credit, socket drain, or request body) before it is reset. |

## Client tuning

Server-side keep-alive and HTTP/2 only pay off if the client keeps and reuses connections.
Most SDK defaults are fine for a handful of threads and too small for an analytics engine
issuing hundreds of concurrent range reads.

### Hadoop s3a (Spark, Hive, Trino with the Hadoop FS)

```xml
<property><name>fs.s3a.endpoint</name><value>https://s3.example.internal</value></property>
<property><name>fs.s3a.path.style.access</name><value>true</value></property>
<!-- pool >= concurrent readers + writers per executor; the default (which differs between
     Hadoop releases) is easily exhausted by vectored reads and parallel uploads -->
<property><name>fs.s3a.connection.maximum</name><value>256</value></property>
<property><name>fs.s3a.threads.max</name><value>128</value></property>
<!-- keep idle connections, but close them before the server / proxy does -->
<property><name>fs.s3a.connection.idle.time</name><value>60s</value></property>
<property><name>fs.s3a.connection.ttl</name><value>5m</value></property>
<property><name>fs.s3a.connection.establish.timeout</name><value>5s</value></property>
<property><name>fs.s3a.connection.timeout</name><value>200s</value></property>
<!-- vectored reads (Parquet/ORC) coalesce nearby ranges into fewer GETs -->
<property><name>fs.s3a.vectored.read.min.seek.size</name><value>128K</value></property>
<property><name>fs.s3a.vectored.read.max.merged.size</name><value>2M</value></property>
```

### AWS SDK for Java v2 (Iceberg `S3FileIO`, custom applications)

```java
S3Client s3 = S3Client.builder()
    .endpointOverride(URI.create("https://s3.example.internal"))
    .forcePathStyle(true)
    .httpClientBuilder(ApacheHttpClient.builder()
        .maxConnections(256)                         // default 50 — too small for parallel scans
        .connectionMaxIdleTime(Duration.ofSeconds(60))  // < server idle timeout (75 s)
        .connectionTimeToLive(Duration.ofMinutes(5))
        .tcpKeepAlive(true))
    .build();
```

For the async client, `NettyNioAsyncHttpClient.builder().maxConcurrency(256)
.connectionMaxIdleTime(Duration.ofSeconds(60))` is the equivalent; with
`.protocol(Protocol.HTTP2)` it multiplexes over HTTP/2 when the endpoint (or the TLS proxy in
front) negotiates `h2`.

Iceberg passes these through catalog properties, e.g. `http-client.type=apache`,
`http-client.apache.max-connections=256`, `http-client.apache.connection-max-idle-time-ms=60000`.

### boto3 / aws-cli

```python
from botocore.config import Config
s3 = boto3.client("s3", endpoint_url="https://s3.example.internal",
                  config=Config(max_pool_connections=64, tcp_keepalive=True,
                                s3={"addressing_style": "path"}))
```

aws-cli: `aws configure set default.s3.max_concurrent_requests 32`.

### Checking that reuse happens

```bash
# HTTP/1.1: two requests, one TCP connection -> "0" connects on the second
curl -s -o /dev/null -o /dev/null -w '%{http_code} %{num_connects}\n' "$URL1" "$URL2"

# HTTP/2 prior knowledge, 20 requests multiplexed on one connection
curl -s --http2-prior-knowledge --parallel -w '%{http_version} %{num_connects}\n' -o /dev/null "$URL" ...

# On an API node: established client connections on the S3 port stay few and stable under load
ss -tan state established '( sport = :8080 )' | wc -l
```

A growing number of `TIME_WAIT` sockets on the client or proxy host under steady load means
connections are not being reused — check the client pool size and idle timeouts, and the
proxy's upstream keep-alive settings.

## Measured effect

Same build, same hardware (Docker Desktop, 32 client threads, 3,200 GETs of 32 KiB objects
against one API node), the only difference being whether each request opens a new connection
(the pre-keep-alive behaviour, reproduced with `Connection: close`) or reuses one:

| Mode | Throughput | p50 latency | p99 latency |
| --- | --- | --- | --- |
| New connection per request | 2,838 req/s | 9.22 ms | 46.23 ms |
| Keep-alive reuse | **4,590 req/s** (+62%) | **6.64 ms** | **11.44 ms** (−75%) |

This measurement runs over a local loopback with no TLS. Across a real network, and especially through a TLS-terminating proxy, each new connection costs more, so reusing connections matters even more there.

## See also

- [Network & Internal Communication](network.md): the three traffic planes and their ports.
- [Nginx Reverse Proxy](../operations/nginx-proxy.md): TLS termination and the upstream keep-alive pool.
- [S3-Compatible API](s3-compatible-api.md): the operations served over both protocols, including multi-range GET.
- [Performance Optimization](performance.md): how request threads, fetch, and chunk pools interact.
