# Monitoring & Metrics

ShannonStore provides built-in monitoring with Prometheus integration.

- Exposes a `/metrics` endpoint for Prometheus scraping, covering S3 operation counts and latencies.
- Collects system metrics (CPU, memory, disk usage) and S3 API statistics from all nodes at configurable intervals.
- Metrics are stored internally with configurable retention (default: 7 days) and automatic cleanup.
- All stored metrics are encrypted at rest.
