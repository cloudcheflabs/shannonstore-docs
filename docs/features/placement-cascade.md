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

The admin UI **Nodes** page shows the same labels per node and a Topology Summary box (zones / racks / hosts / nodes / disks count + a *rack-distinct feasible* flag for the current EC profile).

## Weighted HRW (heterogeneous disks)

When `shannonstore.api.s3.ec.weighted.placement=true`, the placement strategy switches from plain Rendezvous hashing to **weighted Rendezvous (Schindelhauer-Schomaker)** using each disk's capacity as its weight. A node with 2× the storage gets 2× the expected shards over time. Disabled by default — turn on for clusters with unequal disks.

## Chunk rebalance

When the topology changes (a node added, a label updated), the existing chunk metadata still points at the *old* shard locations. The leader-only `RebalanceService` walks objects, recomputes the desired placement, and migrates chunks whose actual placement differs from the new cascade. Trigger from the admin UI **Rebalance** page or via:

```bash
curl -sS -X POST -H "Authorization: Bearer $TOK" \
  http://api:8888/admin/rebalance/run
curl -sS -H "Authorization: Bearer $TOK" \
  http://api:8888/admin/rebalance | jq .
```

Status JSON shows `enabled`, `cycles`, `objectsScanned`, `chunksMoved`, `lastError`. Tiered objects (storage class != STANDARD) are skipped — their location lives outside this cluster.
