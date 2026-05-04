# Cluster Operations

This page covers running a ShannonStore cluster in steady state: starting and stopping roles, observing the cluster, and the day-two maintenance procedures exposed via the admin UI. For first-time installation see [Installation](../installation/installation.md); for the full property reference see [Configuration Management](../features/configuration.md).

## Process Layout

A ShannonStore deployment is composed of three process types:

- **ZooKeeper ensemble** — used for cluster coordination, leader election, and discovery. The release archive ships an embedded single-node ZooKeeper for evaluation; production deployments use a dedicated ensemble (typically three or five nodes).
- **API Server** — terminates the S3 protocol, owns metadata under HRW + vbucket placement, and orchestrates EC writes/reads. Two or more API Servers may run for high availability; one is elected leader for KMS/IAM ownership.
- **Data Node** — owns chunks on local disks. Hosts the chunk store, KMS provider RocksDB warm cache, and the bitrot scrubber.

All three are managed by shell scripts under `bin/` in the release archive.

## Starting and Stopping

### API Server

```agsl
bin/start-api-server.sh
```

The script applies the JVM options from `conf/jvm.conf`, writes a PID to `bin/api-server-<s3-port>.pid`, and redirects stdout/stderr to `logs/shannonstore-api-<s3-port>.log`. The API Server binds three ports: the S3 port (`shannonstore.api.s3.port`, default `8080`), the admin HTTP port (`shannonstore.api.admin.port`, default `8888`), and the internal NIO port (`shannonstore.nio.port`, default `9000`).

To stop gracefully:

```agsl
bin/stop-api-server.sh
```

### Data Node

```agsl
bin/start-data-node.sh
```

Listens on `shannonstore.nio.port` (default per Data Node) and uses one or more storage directories listed in `shannonstore.data.storage.dirs`. PID file goes to `bin/data-node-<port>.pid`.

```agsl
bin/stop-data-node.sh
```

### ZooKeeper

The bundled embedded ZooKeeper is for evaluation only. In production, point all roles at a managed ensemble via `shannonstore.zk.connect`.

## Cluster Bootstrap and Restart

ShannonStore enforces a deterministic, ZK-coordinated bootstrap on every start and restart:

1. **Each node connects to ZooKeeper** and registers itself under `/s3/discovery/{api,data}` with `ready=false`.
2. **Leader election** completes on the API tier via Curator `LeaderLatch`. Exactly one API Server wins, deterministically — non-leaders identify their role by polling `leaderLatch.getLeader()` rather than relying on a timeout-based fallback.
3. **The leader initialises its state** — KMS keys are generated on first cluster bootstrap, or loaded from local RocksDB on restart. IAM is initialised with default users on first bootstrap, or kept from local RocksDB on restart. The leader publishes `/s3/active-leader` with its endpoints and flips `/s3/leader-ready=true`.
4. **Followers and Data Nodes pull from the leader.** Followers fetch KMS and IAM bundles from the leader API and *overwrite* their local RocksDB — the leader is the single source of truth on every restart. Data Nodes read `/s3/active-leader` and pull KMS from that specific leader.
5. **Each non-leader registers `ready=true`** in its discovery payload once the pull completes.
6. **The leader waits for all nodes to flip ready=true**, then publishes `/s3/cluster-ready=true`. Until that znode is `true`, every API request handler returns `503 Retry-After: 5`. After it flips, traffic flows.

If the current leader is restarted, the surviving followers re-elect, the new leader resets `/s3/cluster-ready=false` for the duration of its own init, and the cycle repeats. This guarantees clients never see partial-readiness state.

## Observability

### Admin UI

Browse `http://<api-host>:<admin-port>/admin` to see the cluster snapshot — leader identity, registered nodes, per-node HRW ownership counts (Topology page → HRW Placement), per-disk capacity and usage, KMS state, and the reconciliation sweep stats. The UI is served by the API Server's admin port; any API Server can serve it.

### Health Endpoint

```agsl
curl http://<api-host>:<admin-port>/admin/health
```

Returns `200 UP` once the node has finished init and the cluster-wide readiness znode is `true`. Returns `503 STARTING` while the node is still bootstrapping. Suitable for container readiness probes.

### Metrics

The admin UI surfaces aggregated cluster metrics. Per-node metrics are collected by the leader every minute and stored in a metrics RocksDB; see [Monitoring &amp; Metrics](../features/monitoring.md) for retention and the underlying schema.

### Logs

Each role writes to `logs/shannonstore-<role>-<port>.log` via Logback. The default appender is async file with size-and-time rolling (3 GiB cap by default). For high-throughput production you can switch to a ring-buffer mode by setting `SHANNONSTORE_LOG_MODE=RING_BUFFER`, which keeps the most recent N lines in memory for live admin-UI tailing without the I/O overhead of file logging.

## Maintenance Operations

The admin UI exposes operational actions under the **Nodes** page. They are all admin-authenticated and proxy to the targeted node.

### Maintenance Mode

Toggling **Maintenance Mode** rejects new S3 writes cluster-wide while allowing reads, in-flight requests to drain, and internal RPCs to continue. Use it before invasive operations like scheduled disk replacement or evacuating a node — but **not** for adding/removing API or data nodes, which HRW handles automatically without a maintenance window.

See [Maintenance Mode](../features/maintenance.md).

### HRW Reconciliation Sweep

After a node is added or removed, vbucket ownership shifts and some keys' previous owners are no longer responsible for them. A periodic reconciliation sweep (off by default; toggle from the admin UI's HRW Placement page) drops those stale local copies after verifying the current primary holds the same-or-newer copy. Cassandra's `nodetool cleanup` equivalent.

See [Metadata Management](../features/metadata-management.md).

### Disk Repair

When a disk fails, the admin UI's **Disk Repair** action scans for chunks affected by the dead disk and reconstructs them from EC parity onto the remaining healthy disks. See [Disk Repair Service](../features/disk-repair.md).

### Bitrot Scrubbing

A background scrubber on each Data Node walks each disk on a configurable interval, recomputes per-shard CRC32C checksums, and triggers EC repair on mismatch. Disabled by default; enable via the admin UI per-node toggle. See [Data Integrity](../features/data-integrity.md).

## Disaster Recovery

**Backup and restore is moving to an external destination** (e.g., a different S3 bucket or object store). The previous in-cluster chunk-based backup mechanism — which stored encrypted IAM, KMS, and metadata snapshots inside Data Nodes — has been removed in favour of this external-target design. The replacement is in active design; for now, deployments should snapshot the leader's KMS and IAM RocksDB out-of-band as part of their existing backup tooling.

The cluster's runtime self-healing properties (EC parity reconstruction, bitrot scrubbing, disk repair, leader re-election) remain unchanged and continue to handle in-flight failures without operator intervention.

## Scaling

- **Adding an API Server** — start a new node pointing at the same ZK ensemble and master key. It joins as a follower, pulls KMS/IAM from the leader, and immediately runs a one-shot bootstrap snapshot pull from every peer to materialise the metadata it now owns under HRW. No maintenance window required. Once the cluster is steady, enable the reconciliation sweep on the older nodes to free metadata they no longer own.
- **Adding a Data Node** — start a new Data Node pointing at the same ZK ensemble. It registers with `ready=false`, pulls KMS from the leader, and becomes available for new chunk writes immediately. New chunks land on the new node according to HRW; existing chunks stay on their original Data Nodes (replacement happens lazily as objects are rewritten or the disk repair flow runs).
- **Removing a node** — drain new writes via Maintenance Mode (optional), stop the process. HRW automatically re-routes new writes to the surviving nodes; existing data on the lost node is reconstructible from EC parity (data) or replicated copies (metadata, when R ≥ 2).

## Upgrades

ShannonStore supports rolling upgrades for compatible versions. JSON-serialised state (`ObjectMetadata`, `NodePayload`, IAM snapshots) carries `@JsonIgnoreProperties(ignoreUnknown=true)`, so an old node can read a newer node's payload without deserialisation errors. The recommended upgrade sequence:

1. Upgrade Data Nodes one at a time. After each restart, wait for the node's `ready=true` signal before continuing.
2. Upgrade API followers one at a time. Each upgraded follower will pull fresh KMS/IAM from the (still-old) leader on restart.
3. Upgrade the leader last. It steps down, a follower takes over, and the upgraded process rejoins as a follower.

Across the upgrade, `/s3/cluster-ready` will briefly flip false during each leader transition; clients see short `503 Retry-After` windows, not stale data.
