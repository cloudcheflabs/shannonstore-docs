# Cluster Operations

This page covers running a ShannonStore cluster in steady state: starting and stopping roles, observing the cluster, and the day-two maintenance procedures exposed via the admin UI. For first-time installation see [Installation](../installation/installation.md); for the full property reference see [Configuration Management](../features/configuration.md).

## Process Layout

A ShannonStore deployment is composed of three process types:

- **ZooKeeper** — provides cluster coordination and discovery. The release archive ships an embedded single-node ZooKeeper for evaluation; production deployments use a dedicated ensemble.
- **API Server** — terminates the S3 protocol, holds and replicates metadata, and orchestrates erasure-coded reads and writes. Two or more API Servers run for high availability; one is elected leader for IAM and KMS ownership.
- **Data Node** — stores object shards on local disks. Multiple disks per node are supported.

All three are managed by shell scripts under `bin/` in the release archive.

## Starting and Stopping

### API Server

```
bin/start-api-server.sh
bin/stop-api-server.sh
```

The API Server listens on three ports: the S3 port (default 8080), the admin port (default 8888), and an internal cluster port. Configure them and other runtime options in `conf/shannonstore.properties`; logs and PID files land under `logs/` and `bin/`.

### Data Node

```
bin/start-data-node.sh
bin/stop-data-node.sh
```

Configure the storage directories in `conf/shannonstore.properties` — multiple paths give multi-disk capacity in a single node.

### ZooKeeper

The bundled embedded ZooKeeper is for evaluation only. In production, point all roles at a managed ensemble.

## Cluster Bootstrap and Restart

Bootstrap is fully automatic and deterministic:

1. The cluster elects a leader from the API tier. After a restart, the previous leader tends to win re-election (sticky leadership) so cluster state moves as little as possible.
2. The leader initialises its own state. Default `admin/admin` credentials are created **only on the very first cluster bootstrap** — never again, even if the leader's local state somehow looks empty.
3. Followers and Data Nodes wait until the leader is ready, then synchronise IAM and KMS from the leader (the single source of truth) and report themselves ready.
4. The leader waits until every API node is ready **and at least one data node is registered and ready**, then opens the cluster to client traffic.

Until the cluster is open, S3 and admin requests are rejected with `503 Retry-After: 5`. The wait has no timeout — a cluster brought up without any data nodes simply stays closed until one comes up. Health and login endpoints stay reachable throughout so operators can probe and sign in.

If the leader restarts, surviving followers re-elect (with sticky-leader bias) and the cycle repeats. Clients never see a half-initialised cluster.

### Runtime synchronisation

After bootstrap, every IAM or KMS change runs on the leader and is pushed to all other nodes immediately. Admin requests that mutate IAM (login, password change, access keys, bucket create/delete, STS) automatically route to the leader if they land on a follower.

## Observability

### Admin UI

Browse the admin port at `/admin` to see the cluster snapshot — leader identity, registered nodes, per-node ownership distribution, per-disk capacity and usage, KMS state, and reconciliation sweep stats. Any API Server can serve the UI.

### Health Endpoint

`/admin/health` returns 200 when the node is fully ready, 503 otherwise. Suitable for container readiness probes.

### Metrics & Logs

The admin UI surfaces aggregated cluster metrics; see [Monitoring & Metrics](../features/monitoring.md). Each role writes to a rolling log file under `logs/`; high-throughput deployments can switch to a memory-buffered mode for live admin-UI tailing without disk I/O.

## Maintenance Operations

The admin UI exposes operational actions under the **Nodes** page. They are all admin-authenticated and proxy to the targeted node.

### Maintenance Mode

Toggling **Maintenance Mode** rejects new S3 writes cluster-wide while allowing reads, in-flight requests to drain, and internal RPCs to continue. Use it before invasive operations like scheduled disk replacement or evacuating a node — but **not** for adding/removing API or data nodes, which the cluster handles automatically without a maintenance window.

See [Maintenance Mode](../features/maintenance.md).

### Reconciliation Sweep

After a node is added or removed, ownership shifts and some keys' previous owners are no longer responsible for them. A periodic sweep (off by default; toggle from the admin UI) drops those stale local copies after verifying the current owner has the same-or-newer copy. Equivalent to Cassandra's `nodetool cleanup`.

### Disk Repair

When a disk fails, the admin UI's **Disk Repair** action scans for affected shards and reconstructs them from parity onto the remaining healthy disks. See [Disk Repair Service](../features/disk-repair.md).

### Bitrot Scrubbing

A background scrubber walks each disk on a configurable interval, verifies shard checksums, and triggers parity-based repair on mismatch. Disabled by default; enable per-node from the admin UI. See [Data Integrity](../features/data-integrity.md).

## Disaster Recovery

**Backup and restore is moving to an external destination** (e.g., a different S3 bucket). The previous in-cluster backup mechanism has been removed. For now, snapshot the leader's IAM and KMS state out-of-band as part of your existing backup tooling.

Runtime self-healing — parity reconstruction, bitrot scrubbing, disk repair, leader re-election — continues to handle in-flight failures without operator intervention.

## Scaling

- **Adding an API Server** — start a new node pointing at the same ZooKeeper and master key. It joins as a follower, synchronises IAM/KMS from the leader, and immediately picks up the metadata it now owns. No maintenance window required. Once the cluster is steady, enable the reconciliation sweep on older nodes to free metadata they no longer own.
- **Adding a Data Node** — start a new Data Node pointing at the same ZooKeeper. It becomes available for new chunk writes immediately. Existing chunks stay on their original disks.
- **Removing a node** — optionally drain new writes via Maintenance Mode, then stop the process. The cluster automatically re-routes new writes to the survivors; data on the lost node is reconstructible from parity (data) or replicated copies (metadata).

## Upgrades

ShannonStore supports rolling upgrades for compatible versions. The recommended sequence:

1. Upgrade Data Nodes one at a time, waiting for each to report ready.
2. Upgrade API followers one at a time.
3. Upgrade the leader last; it steps down, a follower takes over, and the upgraded process rejoins as a follower.

Across the upgrade, the cluster briefly closes during each leader transition; clients see short 503 windows, never stale data.
