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

Each API node serves `/metrics` on the admin port (`:8888` by default) in the standard text exposition format. Enable / disable with `shannonstore.api.prometheus.enabled = true|false` (default `true`). **Data nodes do not expose `/metrics` or any HTTP surface at all** — a Data node's only listener is the internal NIO port (`shannonstore.nio.port`); there is no `AdminHandler` equivalent, no HTTP server, and no `/metrics` route in `shannonstore-data`. Data-node system/disk stats reach Prometheus only indirectly, via whatever an API node's `/metrics` endpoint exposes about the cluster (see below).

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
```

The endpoint requires no authentication — it's safe to scrape internally and behind any reverse proxy that hides it from the public internet.

### Metric inventory

This is the **complete** set of custom (non-JVM) Micrometer meters actually registered in source today — it is much smaller than a MinIO/Ceph-class exporter. Grepping the whole codebase for `Counter.builder(`, `Gauge.builder(`, `FunctionCounter.builder(`, and `MetricsRegistry.getTimer(` turns up exactly two families: the S3 request timer, and the object-event-notification counters/gauge below. There is currently **no** per-disk, per-connection-pool, bitrot-scrubber, RocksDB-index, or replication-lag Prometheus metric — those numbers are visible in the Admin UI (sourced from the internal metrics store's periodic snapshot, see "What's collected" below) but are not exported as individually named `/metrics` series.

#### S3 dataplane

| Metric | Type | Labels |
| --- | --- | --- |
| `s3.api.requests` (Prometheus renders as `s3_api_requests_seconds_*`) | timer | `method=PutObject|GetObject|…` |

This is the only S3-request-path metric — there is no separate error counter, bytes-in/out counter, per-verb duration histogram, or multipart-in-flight gauge; error and byte-count breakdowns are only available through the internal metrics store's `s3_api` snapshot section (see "What's collected"), not as their own Prometheus series.

#### Object event notifications

Per-node counters for the [event-notification dispatcher](event-notifications.md).
Because each node runs its own dispatcher, sum across nodes for a cluster total.

| Metric | Type | Meaning |
| --- | --- | --- |
| `shannonstore_notification_emitted_total` | counter | events accepted onto the dispatch queue |
| `shannonstore_notification_delivered_total` | counter | events successfully delivered to a target |
| `shannonstore_notification_dropped_total` | counter | events dropped because the queue was full (overload) |
| `shannonstore_notification_failed_total` | counter | deliveries that failed after all retries |
| `shannonstore_notification_retried_total` | counter | delivery retry attempts |
| `shannonstore_notification_queue_depth` | gauge | current dispatch queue depth |

A useful alert is a sustained drop rate — it means a target is down and the queue is
overflowing:

```promql
rate(shannonstore_notification_dropped_total[5m]) > 0
```

#### Process / JVM

Standard Micrometer JVM bindings: `jvm.memory.used`, `jvm.gc.pause`, `process.cpu.usage`, `system.load.average.1m`, etc. — every Grafana dashboard built for a Java service works unchanged.

### Suggested alerts

Given how small the current Prometheus surface is, the one alert that's directly
actionable from `/metrics` today is request-volume/latency drift on `s3_api_requests`:

```promql
# Sustained drop in request throughput for a given S3 method — could mean the
# method is erroring out upstream of the timer, or clients have stopped calling it.
rate(s3_api_requests_seconds_count[5m])

# p99 latency creeping up by method.
histogram_quantile(0.99, sum(rate(s3_api_requests_seconds_bucket[5m])) by (le, method))

# Object-event notification backlog — a sustained drop rate means a target is
# down and the dispatch queue is overflowing (see below).
rate(shannonstore_notification_dropped_total[5m]) > 0
```

Error rates, disk-repair backlog, and connection-pool exhaustion are all visible in the
Admin UI (sourced from the internal metrics store) but are **not** currently exported as
their own named Prometheus series — there is no `s3.api.errors`, `disk.repair.*`, or
`nio.connection.pool.*` meter in source, despite those names appearing plausible for a
store of this shape.

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

Every interval, the leader pulls a snapshot from every node (API and Data alike) over
the internal NIO protocol (`TYPE_METRICS_REQ`/`TYPE_METRICS_RES`, KMS-encrypted in
transit) via `MetricsRegistry.getMetricsSnapshot()`. What that snapshot actually
contains today:

| Group | Fields |
| --- | --- |
| System | process CPU usage, JVM heap used/max, live thread count |
| Disk | per configured storage-dir: canonical path, total/used/available bytes, usage % |
| S3 ops | per-method request count, mean latency, max latency (from the `s3.api.requests` timer) |
| Raw Prometheus scrape | the node's full `/metrics` text exposition, embedded verbatim under `prometheus_data` |

Network (bytes in/out, drops, retransmits) and cluster-level fields (leader id,
membership, replication lag) are **not** part of this snapshot — there is no code path
that samples them. The same surface backs the Admin UI's overview and per-node detail
panes; aggregation is done by `MetricsCollectorService` on the leader (see below), not
pre-computed per-sample.

## Cluster-wide aggregation

A `MetricsCollectorService` runs only on the leader (`collectAll()` is a no-op on followers) and, on every collection tick, pulls a `TYPE_METRICS_REQ` snapshot from every registered API and Data node and stores each one as its own row — `RocksDBMetricsStore`'s key is `[timestamp][nodeId][metricName]` with `metricName` always the literal string `"all"` (one blob per node per tick). There is no separate pre-computed "cluster-aggregate" row stored under a distinct prefix; any cluster-wide totals the Admin UI shows are computed by summing/averaging the per-node snapshots at query time, not read back from a pre-aggregated record.

If the leader changes, the new leader simply starts pulling on its own next collection tick — no data is lost across the handoff because each node's local `MetricsManager` store already has its own raw samples going back through the retention window.

## Operational guidance

- **Always scrape Prometheus** if you have any external observability stack — the internal store is convenient but Prometheus + Grafana is what production runs against.
- **Don't rely on the internal store as the only source of truth**. It's a self-contained convenience; an operator who needs cross-cluster correlation, long-term retention, or 9s SLO calculations runs Prometheus + a time-series database (VictoriaMetrics, M3, Mimir) behind it.
- **Watch per-disk `usagePercent` in the internal metrics store aggressively** (Admin UI per-node detail pane). Sustained near-full disks are what eventually trigger [Disk Repair](disk-repair.md)'s grace-period reconstruction, and operators want to know before the cluster does.
- **Don't expect a pre-built alert catalog out of the box** — with only `s3.api.requests` and the notification counters exported to Prometheus today, most alerting has to be built against the internal metrics store's snapshot fields or against your own instrumentation layered on top.
- **Tune `metrics.retention.days` based on RocksDB footprint**. The snapshot payload per node per tick is small (system + disk + per-method S3 stats), so 30 days at a 60-second interval is comfortable; 365 days is an order of magnitude larger — only use it if you genuinely need year-old samples.

## See also

- [Configuration Management](configuration.md) — the resolution chain that produces these keys.
- [Cluster Operations](../operations/operations.md) — runbooks built around the alerts above.
- [Maintenance Mode](maintenance.md) — the toggle that briefly stalls metric counters on the dataplane.
- [Encryption & Key Management](kms.md) — what protects the internal metrics store at rest.
- [Disk Repair Service](disk-repair.md) — repair progress is surfaced via `GET /admin/maintenance/repair/status`, not a Prometheus metric.
