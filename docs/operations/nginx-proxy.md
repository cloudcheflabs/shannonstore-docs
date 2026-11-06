# Nginx Reverse Proxy

A production ShannonStore cluster runs multiple API Servers, and each one exposes both the S3 port (default 8080) and the Admin port (default 8888). Putting **nginx in front** gives clients one stable entry point, load-balances across API Servers, and keeps in one place the transport settings that matter for an object store.

This page gives a tuned configuration, says why each setting is there, and shows what it measured.

## Why an nginx ingress

- **Single endpoint.** Clients (S3 SDKs, browsers, monitoring) hit one host:port. You can add or replace API Servers without changing client config.
- **High availability.** nginx spreads requests across API Servers and takes a failing one out of rotation.
- **Two ports unified.** The same listener routes `/admin` and `/metrics` to the Admin upstream and everything else to the S3 upstream, so clients don't track two ports.
- **TLS termination.** ShannonStore serves plain HTTP. nginx (or HAProxy) terminates TLS and can serve HTTP/2 to clients.
- **Large-upload friendly.** Distribution defaults cap request bodies at 1 MB and buffer whole bodies to disk, which breaks S3 multipart uploads of large objects. The configuration below removes both.

## Key directives and why they matter for S3

| Directive | Setting | Why |
|---|---|---|
| `client_max_body_size` | **`0`** (unlimited) | S3 PUT / UploadPart bodies can be many GB. The nginx default (1 MB) rejects them with `413 Request Entity Too Large`. **This is the critical one.** |
| `proxy_request_buffering` / `proxy_buffering` | `off` | Stream bodies through in both directions instead of spooling them to disk. This keeps large transfers moving and bounds nginx memory and disk use. |
| `proxy_buffer_size` (+ `proxy_buffers 8 16k`) | `16k` | Upstream response **headers** must fit in this one buffer even with buffering off. S3 responses with many `x-amz-meta-*` headers overflow the default (4k/8k), and nginx then answers `502` ("upstream sent too big header"). nginx requires `proxy_buffers` to grow along with it. |
| `large_client_header_buffers` | `4 32k` | Presigned URLs (long query strings), SigV4 `Authorization` headers, and object keys of up to 1024 bytes, which roughly triple in size when percent-encoded. |
| `proxy_set_header Host $http_host` | — | **SigV4 signs the Host header.** Forward it exactly as the client sent it, port included, or every request fails with `SignatureDoesNotMatch`. |
| `proxy_http_version 1.1` + `Connection ""` | — | Required for upstream keep-alive. Without them nginx sends `Connection: close` and opens a new connection to ShannonStore for every request. |
| upstream `keepalive` / `keepalive_requests` / `keepalive_timeout` | `256` / `10000` / `60s` | The idle connection pool to the S3 port, per worker process. See [Upstream connection reuse](#upstream-connection-reuse). |
| `least_conn` | — | S3 traffic mixes multi-GB streams with tiny requests. Round-robin keeps sending new requests to a server that is already busy with long transfers; `least_conn` does not. |
| `max_fails=3 fail_timeout=10s` + `proxy_connect_timeout 5s` | — | A dead API server is skipped after a 5 s connect failure instead of the 60 s default, and stays out of rotation for 10 s. |
| `proxy_next_upstream error timeout` + `_tries 2` | — | Retry on another server only when the connection could not be made. nginx never retries POST, and it cannot replay a body it has already streamed. |
| `proxy_send_timeout` / `proxy_read_timeout` / `send_timeout` | `3600s` | Idle time allowed **between two reads or writes**, not a limit on the whole transfer. Slow clients on huge objects need headroom. |
| `access_log … buffer=64k flush=5s` | — | One log write per 64 KiB instead of one per request. This is measurable at tens of thousands of small requests per second. |
| `keepalive_requests` (client side, `http {}`) | `10000` | Requests per client connection (default 1000). SDK connection pools otherwise reconnect needlessly. |
| `worker_connections` / `worker_rlimit_nofile` | `16384` / `65536` | Each proxied request holds two sockets (client and upstream), and the idle upstream pool holds more. The stock `1024` runs out under a busy analytics cluster. |

## Reference configuration

The configuration has two files:

- the process-level settings in `nginx.conf`;
- the ShannonStore part as a `conf.d` file.

Both are the files the ShannonStore e2e and benchmark suites run against (`tests/nginx-main.conf` and `tests/nginx.conf` in the repo).

### `/etc/nginx/nginx.conf`

```nginx
user  nginx;
worker_processes  auto;              # one worker per CPU core
# Every proxied request holds two sockets (client + upstream), plus the idle upstream pool.
# Keep this at least 2 x worker_connections.
worker_rlimit_nofile  65536;

error_log  /var/log/nginx/error.log notice;
pid        /run/nginx.pid;

events {
    worker_connections  16384;       # per worker; client + upstream sockets both count
    multi_accept        on;          # drain the accept queue in one go under connection bursts
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for" '
                      'rt=$request_time urt=$upstream_response_time';
    # Buffered access log: one write per 64 KiB or 5 s instead of one per request.
    access_log  /var/log/nginx/access.log  main  buffer=64k flush=5s;

    sendfile        on;
    tcp_nopush      on;
    tcp_nodelay     on;
    server_tokens   off;

    # Client-side keep-alive (SDK -> nginx). SDK connection pools reuse these connections.
    keepalive_timeout   65s;
    keepalive_requests  10000;       # default 1000

    include /etc/nginx/conf.d/*.conf;
}
```

### `/etc/nginx/conf.d/shannonstore.conf`

```nginx
# --- Request/response handling for an object store ---
client_max_body_size 0;                 # S3 PUT / UploadPart bodies can be many GB
proxy_request_buffering off;            # stream bodies through, both directions
proxy_buffering off;
proxy_buffer_size 16k;                  # upstream response headers (many x-amz-* headers)
proxy_buffers 8 16k;                    # must grow with proxy_buffer_size (nginx -t checks)
large_client_header_buffers 4 32k;      # presigned URLs, SigV4 headers, long encoded keys

# WebSocket upgrade only when the client asks; otherwise an empty Connection header so the
# upstream connection stays in the keep-alive pool.
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      '';
}

# --- 1. S3 API upstream ---
upstream shannon_s3 {
    least_conn;                         # long streams and tiny requests mixed
    server 10.0.0.11:8080 max_fails=3 fail_timeout=10s;
    server 10.0.0.12:8080 max_fails=3 fail_timeout=10s;
    # Keep-alive pool, PER worker process. Must come after the balancing directive.
    keepalive 256;
    keepalive_requests 10000;           # requests per upstream connection before recycling
    keepalive_timeout 60s;              # below ShannonStore's keepalive.idle.timeout.ms (75 s)
}

# --- 2. Admin API & UI upstream ---
upstream shannon_admin {
    server 10.0.0.11:8888 max_fails=3 fail_timeout=10s;
    server 10.0.0.12:8888 max_fails=3 fail_timeout=10s;
    keepalive 16;
}

server {
    listen 80;
    server_name s3.example.internal;

    proxy_connect_timeout  5s;          # fail over fast when an API server is down
    proxy_send_timeout     3600s;       # idle time between two writes / reads,
    proxy_read_timeout     3600s;       # not a limit on the whole transfer
    send_timeout           3600s;
    proxy_next_upstream         error timeout;
    proxy_next_upstream_tries   2;
    proxy_socket_keepalive on;          # TCP keepalive: notice API servers that vanished

    # SigV4 signs Host: forward it exactly as the client sent it.
    proxy_set_header Host              $http_host;
    proxy_set_header X-Real-IP         $remote_addr;
    proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    # Upstream keep-alive needs HTTP/1.1 and no "Connection: close".
    proxy_http_version 1.1;
    proxy_set_header Connection "";

    # Admin UI + Admin API
    location /admin {
        proxy_pass http://shannon_admin;
        # A location with ANY proxy_set_header inherits none from server {}: repeat them.
        proxy_set_header Host              $http_host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Upgrade           $http_upgrade;
        proxy_set_header Connection        $connection_upgrade;
    }

    # Prometheus metrics
    location /metrics {
        proxy_pass http://shannon_admin;
    }

    # S3 API: everything else
    location / {
        proxy_pass http://shannon_s3;
    }
}
```

!!! warning "Two traps in configurations copied from elsewhere"
    - **`proxy_set_header` in a `location` drops every `proxy_set_header` from the `server` block.** nginx inherits these directives only when the inner level sets none. An `/admin` location that adds `Upgrade` / `Connection` and nothing else silently stops forwarding `Host` and `X-Forwarded-*`. Earlier versions of this page had exactly that.
    - **A fixed `Connection "upgrade"`** on a location turns off upstream keep-alive for every request there. Use the `$connection_upgrade` map shown above.

    Don't pass `Upgrade` to the S3 location at all. A client's `Upgrade: h2c` must not reach the S3 port through nginx.

## Upstream connection reuse

The S3 port keeps HTTP/1.1 connections alive, so nginx forwards each client request over an already-open upstream connection instead of making a new TCP handshake per S3 call. Builds before 2026-11 closed every connection after the response, which made the `keepalive` directive a no-op.

| Directive (in `upstream shannon_s3`) | Value | Notes |
|---|---|---|
| `keepalive` | `256` | Idle connections cached **per worker process**. Size it to peak concurrent S3 requests divided by `worker_processes`. If it is too small, nginx closes connections after each burst and leaves `TIME_WAIT` sockets. |
| `keepalive_requests` | `10000` | Requests per connection before nginx recycles it (default 1000). |
| `keepalive_timeout` | `60s` | Idle time before nginx closes a pooled connection. It **must be shorter** than ShannonStore's `shannonstore.network.http.keepalive.idle.timeout.ms` (75 s), so that nginx, not the server, closes an idle connection. Otherwise nginx can send a request on a connection the server is closing. |

`keepalive_requests` and `keepalive_timeout` inside an `upstream` block need nginx ≥ 1.15.3. Apply changes with `nginx -t && nginx -s reload`.

To check that connections are reused, run this under steady load:

```bash
ss -tan state established '( sport = :8080 )' | wc -l    # on an API node: stays small and stable
ss -tan state time-wait '( dport = :8080 )' | wc -l       # on the nginx host: should not keep climbing
```

## TLS and HTTP/2 to clients

Terminate TLS in nginx and let clients use HTTP/2 to nginx. Add this server next to (or instead of) the plain `listen 80` one. The `proxy_*` settings and locations are the same.

```nginx
server {
    listen 443 ssl;
    http2 on;                            # nginx >= 1.25.1; older: "listen 443 ssl http2;"
    server_name s3.example.internal;

    ssl_certificate     /etc/nginx/certs/s3.crt;
    ssl_certificate_key /etc/nginx/certs/s3.key;
    ssl_protocols       TLSv1.2 TLSv1.3;
    # Resumed sessions skip the full handshake when an SDK reconnects.
    ssl_session_cache   shared:SSL:50m;
    ssl_session_timeout 1d;
    ssl_session_tickets off;
    http2_max_concurrent_streams 256;    # concurrent requests per client connection (default 128)

    # ... the same proxy_* settings, proxy_set_header lines and locations as the port-80 server ...
}
```

nginx always speaks **HTTP/1.1 to upstreams**; it cannot forward HTTP/2. The hop from nginx to ShannonStore is therefore the pooled HTTP/1.1 keep-alive tuned above. To multiplex over HTTP/2 all the way to the API nodes, use a proxy that speaks HTTP/2 to upstreams, such as HAProxy with `server … proto h2` or Envoy. The S3 port serves HTTP/2 (h2c) natively.

See [HTTP Protocols & Client Tuning](../features/http-protocols.md#haproxy-with-http2-to-the-s3-port) for a tuned HAProxy configuration, and for when HTTP/2 to the API nodes is worth it. Measured through nginx, it usually isn't.

## Measured effect of the tuned configuration

**Setup**

- The "previous" configuration is what this page used to show: upstream `keepalive`, but round-robin balancing, the stock `nginx.conf` (`worker_connections 1024`, unbuffered access log, client `keepalive_requests 1000`), and a 60 s connect timeout.
- Each configuration ran in its own nginx 1.31 container in front of the same 2 API servers on Docker Desktop. h2load ran on the compose network with presigned GETs and no TLS, and the client spoke HTTP/1.1 to nginx.
- Values are the average of two 8 s runs.

| Configuration | GET 1 KiB, 64 in flight | GET 64 KiB, 64 in flight | GET 8 MiB, 16 in flight |
|---|---:|---:|---:|
| Previous | 42,411 req/s | 25,389 req/s | 4.5 GB/s |
| **Tuned (this page)** | **50,004 req/s** (+18%) | **35,448 req/s** (+40%) | **7.6 GB/s** (+69%) |
| Tuned, but unbuffered access log | 44,828 (−10% vs tuned) | 31,619 (−11%) | 6.4 GB/s |
| Tuned, but round-robin instead of `least_conn` | 46,742 (−7%) | 33,377 (−6%) | 7.1 GB/s |

The buffered access log and `least_conn` each account for part of the gain. The rest comes from the remaining settings together, which were not separated. The client-side `keepalive_requests 10000` alone, tested separately, showed no difference beyond run-to-run noise in this setup. It still saves a reconnect per 1000 requests per client, which matters more where connections are expensive (TLS).

The 8 MiB runs vary by about ±10% between runs (6.9 and 8.4 GB/s for the tuned configuration). All requests succeeded in every run. Absolute numbers on one machine are not a capacity figure, so compare the rows, not the values.

## Deploying

- **Docker Compose.** Mount both files into an `nginx:alpine` service:
    ```yaml
    nginx:
      image: nginx:alpine
      ports: ["80:80", "443:443"]
      volumes:
        - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
        - ./nginx/shannonstore.conf:/etc/nginx/conf.d/default.conf:ro
        - ./nginx/certs:/etc/nginx/certs:ro
      depends_on: [api-server-1, api-server-2]
    ```
- **Standalone nginx.** Put the `conf.d` file at `/etc/nginx/conf.d/shannonstore.conf`, merge the `nginx.conf` settings into the distribution's main file, raise the service's `LimitNOFILE` (systemd) to match `worker_rlimit_nofile`, and reload (`nginx -t && nginx -s reload`).

Point S3 SDKs at `http(s)://<nginx-host>/` and the Admin UI at `http(s)://<nginx-host>/admin`.

The cluster's [readiness gate](../operations/operations.md#cluster-bootstrap-and-restart) still applies. nginx returns whatever the upstream returns, so a cluster that is not open yet answers `503` through the proxy too.
