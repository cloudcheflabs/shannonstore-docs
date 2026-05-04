# Backup &amp; Restore

!!! info "Redesign in progress"
    The previous in-cluster backup mechanism has been removed. ShannonStore is moving to an external-target design where backups are written to a separate destination (a different bucket, region, or storage system), aligning disaster recovery with industry-standard offsite-backup practice.

This feature is in active design and not yet available. Until it ships, the cluster's runtime self-healing properties cover in-flight failures:

- **Erasure coding parity reconstruction** — see [Erasure Coding](ec.md).
- **Bitrot scrubbing** — see [Data Integrity](data-integrity.md).
- **Disk repair service** — see [Disk Repair Service](disk-repair.md).
- **Automatic leader re-election with deterministic readiness signaling** — see [Cluster Operations](../operations/operations.md).

For deployments that need offsite disaster recovery today, snapshot the leader's identity and key state out-of-band as part of your existing backup tooling.
