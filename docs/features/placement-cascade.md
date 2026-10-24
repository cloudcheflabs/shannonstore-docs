# Topology-Aware Placement Cascade

Erasure-coded shards (data + parity) must land on **distinct failure domains**, otherwise the survive-*m*-losses guarantee silently degrades. ShannonStore's placement strategy is a 6-pass cascade that walks the topology from the widest blast-radius down to the disk.

```text
   Zone   →   Rack   →   Host   →   Node   →   Disk   →   Cyclic reuse
   (Pass A)  (Pass B)  (Pass C)  (Pass D)  (Pass E)  (Pass F — desperation, warns)
```

Each pass picks shards from the HRW-ranked candidate set whose chosen label is *not yet covered* by the prior passes. When a label is absent at a layer (host id not configured, etc.) the cascade falls back **upward** — `hostId == null → nodeId`, `rackId == null → hostKey`, `zoneId == null → rackKey` — so a cluster with no labels behaves identically to the legacy strategy.

## Topology labels

Each node advertises three optional labels in its registration payload:

| Label | Default | Override |
|---|---|---|
| `hostId` | the registration `host` (IP/hostname) | `-Dshannonstore.host.id=...` or `SHANNONSTORE_HOST_ID=...` |
| `rackId` | unset | `-Dshannonstore.rack.id=...` or `SHANNONSTORE_RACK_ID=...` |
| `zoneId` | unset | `-Dshannonstore.zone.id=...` or `SHANNONSTORE_ZONE_ID=...` |

`hostId` defaults to the registration host so an operator running multiple data nodes on the same physical machine gets host-distinct placement **out of the box** — no per-machine configuration required.

Inspect the live topology from any API node:

```bash
curl -sS -H "Authorization: Bearer $TOK" \
  http://api:8888/admin/nodes/data | jq '.[] | {id, hostId, rackId, zoneId}'
```

The admin UI **Topology** page shows the same labels per node (each label marked *auto* when it was derived from the registration host) and a **Topology / Fault Domains** summary: the number of distinct zones, racks and hosts and the number of data nodes, each marked *explicit* when an operator set the label or *fallback* when it was inherited from the level below (zone ← rack ← host ← registration address). The same page lists the ZooKeeper ensemble — see [Cluster Operations](../operations/operations.md#zookeeper-in-the-topology-page).

## Weighted HRW (heterogeneous disks)

When `shannonstore.api.s3.ec.weighted.placement=true`, the placement strategy switches from plain Rendezvous hashing to **weighted Rendezvous (Schindelhauer-Schomaker)** using each disk's capacity as its weight. A node with 2× the storage gets 2× the expected shards over time. Disabled by default — turn on for clusters with unequal disks.

## Chunk rebalance

When the topology changes (a node added, a label updated), the existing chunk metadata still points at the *old* shard locations. The leader-only `RebalanceService` walks objects, recomputes the desired placement, and migrates chunks whose actual placement differs from the new cascade. Trigger from the admin UI **Chunk Rebalance** page or via:

```bash
curl -sS -X POST -H "Authorization: Bearer $TOK" \
  http://api:8888/admin/rebalance/run
curl -sS -H "Authorization: Bearer $TOK" \
  http://api:8888/admin/rebalance | jq .
```

Status JSON shows `enabled`, `leader`, `scanInProgress`, `cyclesCompleted`, `objectsScanned`, `chunksMigrated`, `lastCycleStart`, `lastCycleEnd` and `lastCycleStatus`. Every API node answers `/admin/rebalance` with the leader's status, and `enable` / `run` are forwarded to the leader, so the switch is the one the worker actually reads. Changing it is administrator-only. Tiered objects (storage class != STANDARD) are skipped — their location lives outside this cluster.

A chunk counts as misplaced only when its recorded location (`host:port` and disk path) is not among the disks the cascade picks for it today, so a cycle on a cluster whose topology has not changed migrates nothing. (Builds before 1.0.0 compared against the data node's registry id instead and re-copied every chunk on every cycle.)
