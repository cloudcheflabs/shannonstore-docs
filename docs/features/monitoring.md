# Monitoring & Metrics

ShannonStore exposes two telemetry surfaces side by side: a live Prometheus endpoint for scraping and alerting, and a built-in time-series store for in-product dashboards and historical replay. Both consume the same `MetricsRegistry` so the values stay consistent — Prometheus is for external observability stacks, the internal store is for operators who run the Admin UI and need a self-contained view.

```text
   Every API node + every data node
        │
        ▼
   ┌──────────────────────────────────────┐
   │  MetricsRegistry (per process)       │
   │    - micrometer-style timers,        │
   │      counters, gauges                │
   │    - per-request bucketed latencies  │
   └────────┬─────────────────────────────┘
            │
            ├──► /metrics  (text exposition)   ◄── Prometheus / Grafana / VictoriaMetrics
            │
            └──► MetricsManager                 ◄── Admin UI charts
                  (encrypted RocksDB,
                   retention swept periodically)
```

## Prometheus endpoint

Each API node serves `/metrics` on the admin port (`:8888` by default) in the standard text exposition format. Enable / disable with `shannonstore.api.prometheus.enabled = true|false` (default `true`). Data nodes expose the same endpoint on their admin surface.

A minimal scrape config:

```yaml
scrape_configs:
  - job_name: shannonstore-api
    metrics_path: /metrics
    static_configs:
      - targets:
          - api-1:8888
          - api-2:8888
          - api-3:8888
  - job_name: shannonstore-data
    metrics_path: /metrics
    static_configs:
      - targets:
          - data-1:8889
          - data-2:8889
          - data-3:8889
```

The endpoint requires no authentication — it's safe to scrape internally and behind any reverse proxy that hides it from the public internet.

### Metric inventory

The most useful counters and gauges, by domain. (The exact metric names follow Micrometer conventions: dot-separated identifiers, Prometheus auto-renames dots to underscores.)

#### S3 dataplane

| Metric | Type | Labels |
| --- | --- | --- |
| `shannonstore.s3.api.requests` | timer | `method=PutObject|GetObject|…` |
| `shannonstore.s3.api.errors` | counter | `method`, `code=NoSuchKey|AccessDenied|…` |
| `shannonstore.s3.api.bytes.in` | counter | `method` |
| `shannonstore.s3.api.bytes.out` | counter | `method` |
| `shannonstore.s3.put.duration` | histogram | percentile buckets |
| `shannonstore.s3.get.duration` | histogram | percentile buckets |
| `shannonstore.s3.multipart.in.flight` | gauge | per node |

#### Internal NIO plane

| Metric | Type | Labels |
| --- | --- | --- |
| `shannonstore.nio.connection.pool.idle` | gauge | `peer=host:port` |
| `shannonstore.nio.connection.pool.acquire.timeout.total` | counter | `peer` |
| `shannonstore.nio.fetch.timeout.total` | counter | `peer` |
| `shannonstore.network.compression.bytes.in.total` | counter | `algorithm` |
| `shannonstore.network.compression.bytes.out.total` | counter | `algorithm` |

#### Storage

| Metric | Type | Labels |
| --- | --- | --- |
| `shannonstore.disk.bytes.total` | gauge | `node`, `disk=/mnt/disk-1\|…` |
| `shannonstore.disk.bytes.used` | gauge | `node`, `disk` |
| `shannonstore.disk.available` | gauge (0/1) | `node`, `disk` |
| `shannonstore.disk.chunks.count` | gauge | `node`, `disk` |
| `shannonstore.disk.repair.repairs.total` | counter | `node` |
| `shannonstore.disk.repair.bytes.rebuilt.total` | counter | `node` |
| `shannonstore.bitrot.scrubber.scanned.total` | counter | `node` |
| `shannonstore.bitrot.scrubber.corrupt.total` | counter | `node` |
| `shannonstore.bitrot.scrubber.repaired.total` | counter | `node` |

#### Metadata + replication

| Metric | Type | Labels |
| --- | --- | --- |
| `shannonstore.index.rocksdb.size.bytes` | gauge | `node` |
| `shannonstore.index.rocksdb.gets.total` | counter | `node` |
| `shannonstore.index.rocksdb.puts.total` | counter | `node` |
| `shannonstore.metadata.replication.lag.ms` | gauge | `mode=PULL|SYNC|ASYNC_PUSH` |

#### Process / JVM

Standard Micrometer JVM bindings: `jvm.memory.used`, `jvm.gc.pause`, `process.cpu.usage`, `system.load.average.1m`, etc. — every Grafana dashboard built for a Java service works unchanged.

### Suggested alerts

Three production alerts catch the vast majority of operationally important conditions:

```promql
# High request error rate — anything over 1% sustained for 5 minutes is investigation-worthy.
sum(rate(shannonstore_s3_api_errors[5m])) by (method)
  / sum(rate(shannonstore_s3_api_requests_count[5m])) by (method)
  > 0.01

# Disk repair backlog — repair is normal but a growing backlog signals a node is dying.
increase(shannonstore_disk_repair_repairs_total[1h]) > 1000

# Connection pool exhaustion — acquire timeouts mean a peer is slow or dead.
rate(shannonstore_nio_connection_pool_acquire_timeout_total[5m]) > 0
```

## Internal metrics store

For operators who don't run Prometheus, the API node also collects metrics into its own RocksDB-backed store (`MetricsManager`). The Admin UI's charts read directly from this store, so a fresh install gets useful dashboards immediately with no external system to deploy.

| Key | Default | Effect |
| --- | --- | --- |
| `shannonstore.api.metrics.collection.interval.seconds` | `60` | Sample tick — system + S3 stats are recorded every interval. |
| `shannonstore.api.metrics.retention.days` | `30` | History kept. |
| `shannonstore.api.metrics.retention.cleanup.interval.hours` | `12` | Sweep cadence for evicting samples older than retention. |
| `shannonstore.api.history.retention.days` | `365` | Operation-history (audit-style records) retention. |

The store is on-disk and **KMS-encrypted at rest** through the same envelope-encryption pipeline that protects IAM and object metadata. A copy of the metrics RocksDB extracted out-of-band is unreadable without the cluster's master key.

Retention sweeps run on a background timer; an operator can also force a sweep through the Admin REST when changing retention from a high value to a low one (a no-op default behaviour would otherwise leave the old samples on disk until the next scheduled sweep).

## What's collected

Every interval, each node samples:

| Group | Examples |
| --- | --- |
| System | CPU usage %, load average, free + total memory, page faults |
| Disk | per-disk capacity, used bytes, IO ops/sec, errors |
| Network | bytes in / out, dropped packets, retransmits |
| JVM | heap used, GC pause time, thread counts, open file descriptors |
| S3 ops | counts and durations by method since last sample |
| Cluster | leader id, membership, replication lag |

The same surface backs the Admin UI's overview, per-node detail, and per-bucket throughput panes. Aggregations are pre-computed at sample time so the UI query is cheap (point lookups against a single RocksDB column family).

## Cluster-wide aggregation

A `MetricsCollectorService` running on the leader pulls the per-node samples and produces cluster-aggregate rows (total throughput, cluster-wide capacity, weighted average latencies). These aggregates are what the Admin UI's overview shows and what alert rules typically threshold against; they ship in the same RocksDB store under a distinct prefix.

If the leader changes, the new leader picks up the aggregation work on the next collection tick — no data is lost across the handoff because each follower's local store has the raw samples and the aggregation is recomputed from them.

## Operational guidance

- **Always scrape Prometheus** if you have any external observability stack — the internal store is convenient but Prometheus + Grafana is what production runs against.
- **Don't rely on the internal store as the only source of truth**. It's a self-contained convenience; an operator who needs cross-cluster correlation, long-term retention, or 9s SLO calculations runs Prometheus + a time-series database (VictoriaMetrics, M3, Mimir) behind it.
- **Watch `disk.available == 0` aggressively**. A flap of the disk-health probe is normal; sustained unavailability for more than the repair grace window kicks in reconstruction, and operators want to know before the cluster does.
- **Alert on `shannonstore.nio.connection.pool.acquire.timeout.total`**, not just on `shannonstore.s3.api.errors`. The connection-pool timeout precedes the S3 error by enough seconds to be worth catching early.
- **Tune `metrics.retention.days` based on RocksDB footprint**. 30 days × 60-second samples × ~200 metrics per sample is a few hundred MB per node — comfortable. 365 days is order-of-magnitude larger; only use it if you genuinely need year-old samples.

## See also

- [Configuration Management](configuration.md) — the resolution chain that produces these keys.
- [Cluster Operations](../operations/operations.md) — runbooks built around the alerts above.
- [Maintenance Mode](maintenance.md) — the toggle that briefly stalls metric counters on the dataplane.
- [Encryption & Key Management](kms.md) — what protects the internal metrics store at rest.
- [Disk Repair Service](disk-repair.md) — the source of the repair counters.
