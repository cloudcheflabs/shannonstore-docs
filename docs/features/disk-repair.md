# Disk Repair Service

ShannonStore includes an automatic disk repair system that detects failed disks and reconstructs lost shards using erasure coding.

- Periodically scans metadata to identify shards on disks that are no longer healthy, with a grace period to avoid reacting to transient issues.
- Surviving shards are fetched from healthy nodes, missing shards are reconstructed via erasure coding, and repaired shards are placed on new healthy disks.
- Metadata is updated to reflect the new shard locations and replicated to ensure consistency.
- Configurable scan interval, concurrency limits, and minimum disk space thresholds. Manual trigger is also available via Admin UI or API.
