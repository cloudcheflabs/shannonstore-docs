# Nginx Reverse Proxy

A production ShannonStore cluster runs multiple API Servers — each exposing both the S3 port (default 8080) and the Admin port (default 8888). Putting **nginx in front** gives clients one stable entry point, load-balances across API Servers, and centralises the few transport tweaks that matter for an object store.

## Why an nginx ingress

- **Single endpoint** — clients (S3 SDKs, browsers, monitoring) hit one host:port; you can add or replace API Servers without changing client config.
- **High availability** — nginx upstreams round-robin across api-server instances; an unhealthy server is taken out of rotation automatically.
- **Two ports unified** — the same listener routes `/admin` and `/metrics` to the Admin upstream, and everything else to the S3 upstream — clients don't track two ports.
- **Large-upload friendly** — defaults in distributions cap request bodies at 1 MB and buffer the whole body to disk, which breaks S3 multipart uploads of large objects. The config below removes those caps.

## Key directives — and why they matter for S3

| Directive | Setting | Why |
|---|---|---|
| `client_max_body_size` | **`0`** (unlimited) | S3 PUT / UploadPart can be many GB. The nginx default (1 MB) rejects them with `413 Request Entity Too Large`. `0` disables the limit. |
| `proxy_request_buffering` | `off` | Stream the request body straight to the upstream instead of spooling to disk first — keeps large uploads moving and bounds memory/disk. |
| `proxy_buffering` | `off` | Mirror on the response side — stream downloads through, don't buffer. |
| `proxy_send_timeout` / `proxy_read_timeout` / `send_timeout` | `3600s` | Long transfers (huge objects, slow networks) need long timeouts. |
| `proxy_http_version 1.1` + `Connection ""` | enable upstream keepalive | Required for `keepalive` in the upstream block to do anything. |

The `client_max_body_size 0` is **the critical one** — without it the S3 API is unusable for objects above the nginx default.

## Reference configuration

The cluster's tests/CI ship a ready-to-use config at `tests/nginx.conf` in the ShannonStore repo. The complete file:

```nginx
# --- Unlimited request body size — required for large S3 uploads ---
client_max_body_size 0;

# --- Disable buffering for large upload/download stability ---
proxy_request_buffering off;
proxy_buffering off;

# --- 1. S3 API Upstream ---
upstream shannon_s3 {
    server api-server-1:8080;
    server api-server-2:8080;
    server localhost:8080 backup;
    server localhost:8081 backup;
    keepalive 64;
}

# --- 2. Admin API & UI Upstream ---
upstream shannon_admin {
    server api-server-1:8888;
    server api-server-2:8888;
    server localhost:8888 backup;
    server localhost:8889 backup;
    keepalive 16;
}

server {
    listen 80;
    server_name localhost;

    # Long transfers / slow networks
    proxy_connect_timeout  60s;
    proxy_send_timeout     3600s;
    proxy_read_timeout     3600s;
    send_timeout           3600s;

    # Pass-through request metadata
    proxy_set_header Host              $http_host;
    proxy_set_header X-Real-IP         $remote_addr;
    proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    # Upstream keepalive (HTTP/1.1 required)
    proxy_http_version 1.1;
    proxy_set_header Connection "";

    # Admin UI + Admin API
    location /admin {
        proxy_pass http://shannon_admin;
        # WebSocket upgrade — preserved for any live admin tooling
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }

    # Prometheus metrics
    location /metrics {
        proxy_pass http://shannon_admin;
    }

    # S3 API — everything else
    location / {
        proxy_pass http://shannon_s3;
        # Stream large objects through nginx
        proxy_request_buffering off;
        proxy_buffering off;
    }
}
```

`shannon_s3` carries S3 PUT/GET/multipart traffic; `shannon_admin` carries the Admin UI, the Admin REST API, and the Prometheus `/metrics` endpoint. `localhost:8080-8081` / `localhost:8888-8889` `backup` entries cover a local/single-host deployment where api-server-1/2 hostnames aren't resolvable.

## Deploying

- **Docker Compose** — drop the file at `nginx/nginx.conf` and mount it into an `nginx:alpine` service:
    ```yaml
    nginx:
      image: nginx:alpine
      ports: ["80:80"]
      volumes:
        - ./nginx/nginx.conf:/etc/nginx/conf.d/shannonstore.conf:ro
      depends_on: [api-server-1, api-server-2]
    ```
- **Standalone nginx** — drop it under `/etc/nginx/conf.d/shannonstore.conf` and reload (`nginx -s reload`).

Point S3 SDKs at `http://<nginx-host>/` and the Admin UI at `http://<nginx-host>/admin`. The cluster's [readiness gate](../operations/operations.md#cluster-bootstrap-and-restart) still applies — nginx returns whatever the upstream returns, so a not-yet-open cluster will respond `503` through the proxy too.
