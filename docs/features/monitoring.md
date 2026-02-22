# Monitoring & Metrics

## Prometheus Integration

- /metrics endpoint exports metrics in Prometheus text format for scraping.
- Uses Micrometer registry (PrometheusMeterRegistry) for counters, gauges, and histograms.
- Covers S3 operation counts and latencies by method type.

## Time-Series Metrics Storage

- Metrics stored in a dedicated RocksDB instance with time-series optimized key format: [8-byte timestamp (big-endian)]:[nodeId]:[metricName].
- SNAPPY compression and bloom filters (10 bits) enabled for efficient storage and retrieval.
- Prefix scan support for time-range queries.

## Collected Metrics

- CPU usage: Percentage utilization per node.
- Memory usage: Bytes used per node.
- Disk usage: Per-mount-point capacity, used bytes, and available bytes on Data Nodes.
- S3 API statistics: Per-method (GET, PUT, DELETE, etc.) request count and mean latency.
- Metrics are encrypted with the KMS default key before storage.

## Retention & Cleanup

- Configurable retention period (default: 7 days).
- Automated cleanup every 12 hours removes data older than the retention threshold.
- RocksDB compaction triggered after range deletion for space reclamation.

## Collection Service

- MetricsCollectorService runs on a configurable interval (default: 30 seconds).
- Gathers metrics from all API and Data Nodes.
- Centralized collection on the leader node for aggregated queries.