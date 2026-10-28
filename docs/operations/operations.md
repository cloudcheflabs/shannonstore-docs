# Cluster Operations

This page covers running a ShannonStore cluster in steady state: starting and stopping roles, observing the cluster, and the day-two maintenance procedures exposed via the admin UI. For first-time installation see [Installation](../installation/installation.md); for the full property reference see [Configuration Management](../features/configuration.md).

## Process Layout

A ShannonStore deployment is composed of three process types:

- **ZooKeeper** — provides cluster coordination and discovery. The release archive ships a single-node ZooKeeper (`bin/start-zk.sh`) for evaluation; production deployments use a dedicated ensemble.
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

With no argument, `bin/stop-data-node.sh` stops the Data Node running from this installation. The start script names the PID file after the instance's port (`bin/data-node-<port>.pid`, the NIO port), and the stop script looks there. When several Data Nodes run from the same installation it does not guess: it lists their ports and exits 1, and you pass the port of the one to stop, e.g. `bin/stop-data-node.sh 9001`. It reports "not running" only when no Data Node from this installation is running.
`bin/stop-api-server.sh` works the same way with the API Server's S3 port.

### ZooKeeper

```
bin/start-zk.sh
bin/stop-zk.sh
```

The bundled ZooKeeper runs the standard `QuorumPeerMain` with `conf/zk/zoo.cfg` (client port 2181, data under `data/zookeeper`, log in `logs/zk.log`). It is a single server, for evaluation only. In production, point every API Server and Data Node at a managed ensemble of three or five servers through `shannonstore.zk.connect` (`zk1:2181,zk2:2181,zk3:2181`).

The admin **Topology** page reads each server's status with the `srvr` four-letter command and its server id (`myid`) with `conf`. ZooKeeper 3.5+ answers `srvr` by default; if your ensemble sets `4lw.commands.whitelist`, keep `srvr` in it, and add `conf` — which is **not** on the default whitelist — so the page can show each server's id and warn about a duplicate `myid` (the bundled `zoo.cfg` sets `srvr,ruok,conf`). Without `srvr` the page still shows the server as UP but cannot show its role; without `conf` the id reads `server id unknown (conf not in 4lw.commands.whitelist)`.

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

A node can miss pushes while it stalls longer than its ZooKeeper session timeout, for example during a long GC pause, a frozen VM or a network partition. When it reconnects, it registers again (its log records `Re-registered … the previous session expired`) and catches up. A follower API Server pulls KMS and IAM from the leader, and a Data Node pulls KMS. A KMS key rotated while the node was away is therefore available to it as soon as it is back. If a node receives an IAM push it cannot decrypt, it fetches the leader's KMS once and retries.

A leader that stalls this long loses leadership at once and does not keep acting as leader when it returns. A password change sent to a follower is answered with success only once that follower accepts the new password. Otherwise the answer is `502`.

## Observability

### Admin UI

Browse the admin port at `/admin` to see the cluster snapshot — leader identity, registered nodes, per-node ownership distribution, per-disk capacity and usage, KMS state, and reconciliation sweep stats. Any API Server can serve the UI.

The **Topology** page lists everything the cluster runs on:

- **Fault domains** — distinct zones, racks and hosts among the data nodes (see [Topology-Aware Placement](../features/placement-cascade.md)).
- **API Servers** — node id, address, ports, maintenance flag, logs.
- **Data Nodes** — node id, address, fault-domain labels, per-disk capacity and usage.
- **ZooKeeper** — every server of the ensemble, the quorum, and each API Server's own ZooKeeper session (below).

### ZooKeeper in the Topology page

The ZooKeeper section shows every server named in `shannonstore.zk.connect`, in that order:

| Column | Meaning |
|---|---|
| Server | `host:port` from the connect string (a chroot suffix is ignored; a host without a port is 2181) |
| Server id | the server's own id (`myid`) as it reports it to `conf`, or `server id unknown (...)` with the reason — see [Server ids](#server-ids-myid) |
| Role | `leader`, `follower`, `observer`, or `standalone` for a single server; `unknown` when the server is down or does not answer `srvr` |
| Version | the ZooKeeper server version |
| Connections | client connections open on that server |
| Latency min/avg/max | request latency the server measured, in ms |
| Zxid / Znodes | the last transaction id the server applied, and its znode count |
| Status | UP, or DOWN with the reason (connection refused, timeout, ...) |

The header shows `UP / total` and the quorum: **Quorum OK** when a strict majority of the ensemble is up and one server is the leader (a standalone server is its own quorum), **No Quorum** otherwise. Without quorum ZooKeeper refuses writes, so leader election, node registration and the cluster-ready flag stop working until a majority is back.

Below the table, **API node sessions** show each API Server's own ZooKeeper session: its connection state as Curator last reported it (`CONNECTED`, `RECONNECTED`, `SUSPENDED`, `LOST`, `READ_ONLY`), the server the session is attached to, the session id and the negotiated session timeout. A `SUSPENDED` session means that node lost its server and is trying another; `LOST` means the session expired and the node's ephemeral registrations were removed.

How it works: the API Server that answers the page asks every ZooKeeper server in parallel with `srvr` on the client port, with a timeout of `shannonstore.api.admin.zk.probe.timeout.ms` (default 2000 ms), so one dead server costs one timeout. The page refreshes every 5 seconds. The same data is available from the admin REST API (admin login required):

```bash
# the ensemble, the quorum and the answering node's session
curl -s -H "Authorization: Bearer $TOK" http://api-1:8888/admin/cluster/zookeeper | jq

# answered by another API Server, to read that node's session
curl -s -H "Authorization: Bearer $TOK" \
  "http://api-1:8888/admin/cluster/zookeeper?targetHost=api-2&targetPort=8888" | jq .client
```

```json
{
  "connectString": "zk1:2181,zk2:2181,zk3:2181",
  "servers": [
    { "host": "zk1", "port": 2181, "status": "UP", "version": "3.9.1", "latencyMin": 0, "latencyAvg": 0.4,
      "latencyMax": 7, "connections": 6, "outstanding": 0, "zxid": "0x10000004a", "mode": "follower", "nodeCount": 61,
      "probeMs": 3, "serverId": 1, "serverIdNote": null,
      "membership": { "1": "zk1:2888:3888:participant;0.0.0.0:2181", "2": "zk2:2888:3888:participant;0.0.0.0:2181",
                      "3": "zk3:2888:3888:participant;0.0.0.0:2181" } },
    { "host": "zk2", "port": 2181, "status": "UP", "mode": "leader", "serverId": 2, "...": "..." },
    { "host": "zk3", "port": 2181, "status": "DOWN", "mode": "unknown", "error": "ConnectException: Connection refused",
      "serverId": null, "serverIdNote": "server id unknown (server is down)", "membership": {} }
  ],
  "quorum": { "up": 2, "total": 3, "required": 2, "hasQuorum": true, "leader": "zk2:2181" },
  "warnings": [ "ZooKeeper server id unknown on zk3:2181 (server is down) — a duplicate myid cannot be ruled out" ],
  "client": { "nodeId": "ss-api-1", "state": "CONNECTED", "connected": true, "sessionId": "0x100000a3c0b0002",
              "sessionTimeoutMs": 40000, "connectedServer": "zk1/172.18.0.3:2181" },
  "checkedAt": 1760780000000
}
```

Data Nodes use the same ensemble but have no HTTP surface, so their sessions are not listed; a Data Node that lost its session disappears from the **Data Nodes** table when its ephemeral registration expires.

`warnings` is empty when every server reported a unique id and all of them agree on the membership.

#### Server ids (myid)

Every server of an ensemble needs its own id (`myid`). Two servers with the same id — typically
left behind by a scale-out or a rebuild that reused an instance number — or servers that do not
agree on who is in the ensemble break it **silently**: ZooKeeper raises no error, and clients just
see an unstable or split service. The section therefore also asks each server that answers with the
`conf` four-letter word, which reports the server's own `serverId` and the ensemble membership it
was configured with (its `server.N=` lines). The **Server id** column shows the id, or
`server id unknown (...)` with the reason, and a red line above the table names each problem:

| Warning | Meaning |
|---|---|
| `Duplicate ZooKeeper server id 2 on zk2:2181, zk3:2181 — the ensemble is misconfigured; give every server a unique myid` | two or more configured servers report the same id |
| `ZooKeeper server zk4:2181 reports server id 4, which is not in its own ensemble membership (1, 2, 3) — check its myid` | a server's id is not among the `server.N` entries it was configured with |
| `ZooKeeper servers disagree about the ensemble membership: zk1:2181, zk2:2181 list server ids 1, 2, 3; zk3:2181 lists server ids 2 — every server must be configured with the same server.N list` | the servers were configured with different ensembles |
| `ZooKeeper server id unknown on zk1:2181, zk2:2181, zk3:2181 (conf not in 4lw.commands.whitelist) — a duplicate myid cannot be ruled out` | the id could not be read (here `conf` is not whitelisted; also `server is down`, `server is not serving requests`) — the check is never reported as passing for those servers |
| `ZooKeeper servers zk1:2181, zk2:2181 run standalone — a standalone server is a separate single-server ZooKeeper, not part of one ensemble with the other configured servers` | the connect string lists several servers but some of them run standalone |

A healthy ensemble shows each server's id and no warning; a single standalone ZooKeeper reports id
`0` and is not a warning. To fix a duplicate or a mismatch, correct the `myid` file in the server's
data directory (`ZOO_MY_ID` for the `zookeeper` Docker image) and its `server.N=` lines so that
every server has a unique id and all of them list the same ensemble, then restart that server.

`conf` is **not** on ZooKeeper's default whitelist. Add it next to `srvr`, for example
`4lw.commands.whitelist=srvr,conf` (`ZOO_4LW_COMMANDS_WHITELIST=srvr,conf` for the Docker image).
Without it every server's id reads `server id unknown (conf not in 4lw.commands.whitelist)`.
The ZooKeeper bundled with ShannonStore (`conf/zk/zoo.cfg`) whitelists `srvr,ruok,conf`. **Quorum OK** says
nothing about the ids: an ensemble with a duplicate id can still show a quorum.

`tests/e2e-zk-myid.sh` verifies all of this on a real three-server ensemble, on the API and on the
Topology page from a non-secure origin (`tests/ui-e2e/45-zk-myid.mjs`).

### Health Endpoint

`/admin/health` returns 200 when the node is fully ready, 503 otherwise. Suitable for container readiness probes.

### Metrics & Logs

The admin UI surfaces aggregated cluster metrics; see [Monitoring & Metrics](../features/monitoring.md). Each role writes to a rolling log file under `logs/`; high-throughput deployments can switch to a memory-buffered mode for live admin-UI tailing without disk I/O.

## Maintenance Operations

The admin UI exposes operational actions under the **Nodes** page. They are all admin-authenticated and proxy to the targeted node.

### Maintenance Mode

Toggling **Maintenance Mode** refuses **every** S3 request cluster-wide — reads as well as writes, authenticated or anonymous — with `503` and a `Retry-After`. Requests already in flight run to completion, internal RPCs continue, and background work (disk repair, scrubber, rebalance, lifecycle) keeps running: closing the cluster to clients is what gives that work a quiet window.

Data-changing admin routes are refused too, so the object browser cannot delete objects behind your back. Admin reads, settings edits and the maintenance switch itself stay open.

The setting is stored, not held in memory, so an API node restarted mid-window comes back still refusing traffic.

Use it before invasive operations like scheduled disk replacement or evacuating a node — but **not** for adding or removing API/data nodes, which the cluster handles on its own without a window.

See [Maintenance Mode](../features/maintenance.md).

### Reconciliation Sweep

After a node is added or removed, ownership shifts and some keys' previous owners are no longer responsible for them. A periodic sweep (off by default; toggle from the admin UI) drops those stale local copies after verifying the current owner has the same-or-newer copy. Equivalent to Cassandra's `nodetool cleanup`.

### Disk Repair

When a disk fails, the admin UI's **Disk Repair** action scans for affected shards and reconstructs them from parity onto the remaining healthy disks. See [Disk Repair Service](../features/disk-repair.md).

### Bitrot Scrubbing

A background scrubber walks each disk on a configurable interval, verifies shard checksums, and triggers parity-based repair on mismatch. Disabled by default; enable per-node from the admin UI. See [Data Integrity](../features/data-integrity.md).

## Disaster Recovery

ShannonStore ships an in-cluster S3 Backup mechanism — content-addressed, incremental, parallel-fan-out to an external S3-compatible destination bucket (another ShannonStore, MinIO, or AWS S3), configured and triggered from the Admin UI or the `/admin/backup/*` REST endpoints, with a staged restore that swaps in atomically on restart. See [Backup & Restore](../features/backup.md) for the full mechanism, scheduling, and restore procedure.

Runtime self-healing — parity reconstruction, bitrot scrubbing, disk repair, leader re-election — continues to handle in-flight failures without operator intervention and is not a substitute for the offsite backup above.

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
