# Backup &amp; Restore

!!! info "Redesign in progress"
    The previous in-cluster backup mechanism — which stored encrypted IAM, KMS, and metadata snapshots inside Data Nodes as `_system_*` chunks — has been removed. ShannonStore is moving to an external-target design where backups are written to a separate S3 destination (a different bucket, region, or storage system), aligning disaster recovery with industry-standard offsite-backup practice.

This feature is in active design and not yet available. Until it ships, the cluster's runtime self-healing properties cover in-flight failures:

- **EC parity reconstruction** — see [Erasure Coding](ec.md).
- **Bitrot scrubbing** — see [Data Integrity](data-integrity.md).
- **Disk repair service** — see [Disk Repair Service](disk-repair.md).
- **Leader re-election with deterministic cluster-ready signaling** — see [Cluster Operations](../operations/operations.md).

For deployments that need offsite disaster recovery today, snapshot the leader's KMS and IAM RocksDB out-of-band as part of your existing backup tooling.
