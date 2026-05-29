# Multi-Disk Storage

ShannonStore data nodes are JBOD — Just a Bunch Of Disks. Each data node holds one or more storage roots configured via `shannonstore.data.storage.dirs`, and the data node treats each root as an independent failure domain. A disk dies → only the shards living on that disk need re-placement; the rest of the disks keep serving normally without a rebuild.

```text
   shannonstore.data.storage.dirs=/mnt/disk-1,/mnt/disk-2,/mnt/disk-3,/mnt/disk-4

   Data node:
     ├── /mnt/disk-1   (chunk files, ChunkStore index for this disk)
     ├── /mnt/disk-2   (chunk files, …)
     ├── /mnt/disk-3   (chunk files, …)
     └── /mnt/disk-4   (chunk files, …)

   API tier sees:
       node-X / disk-1, node-X / disk-2, node-X / disk-3, node-X / disk-4
       — four placement targets per node
```

The data node does not stripe a single shard across disks. Each shard lands on **exactly one** disk and stays there for its lifetime (or until repair re-places it). Cross-disk parity is not the data node's job — that's already handled by the cluster-level erasure coding in the API tier (see [Erasure Coding](ec.md)).

## Why JBOD over RAID

A RAID-5 / RAID-6 underlayer per data node would add a second layer of erasure coding on top of the cluster's already-existing EC, doubling the storage overhead for the same logical failure tolerance. JBOD lets ShannonStore allocate every byte of disk to either *data* or *cluster parity*, never to *local parity*. The trade-off is that a single disk dying loses all of its shards immediately — but the cluster's EC handles exactly that case, and the rebuild is faster (and finer-grained) than a RAID rebuild.

| Layout | Storage overhead | Rebuild scope on single-disk failure |
| --- | --- | --- |
| ShannonStore JBOD + cluster EC (k=4 m=2) | 50% (cluster level) | only the shards on the dead disk |
| RAID-5 + cluster EC (k=4 m=2) | 50% × 1.25 = 62.5% | the whole RAID group, then nothing at the cluster level |
| RAID-6 + cluster EC (k=4 m=2) | 50% × 1.5 = 75% | the whole RAID group |
| 3× replication, no EC | 200% | one of three full copies |

So the recommended deployment is bare ext4 / xfs filesystems mounted at distinct paths, one per disk, with no RAID below.

## Disk placement on a node

When a write arrives at a data node it picks a target disk for the shard using a simple hash:

```
disk_index = HRW(chunkId, available_disks)
```

Same key+shardIndex pair maps to the same disk deterministically — so a reader, given the metadata's `diskPath`, finds the file without a lookup table. The algorithm respects the *currently healthy* disk set: a disk that fails health probes drops out of the placement set and new shards skip it. Shards already on the failed disk are picked up by the [Disk Repair Service](disk-repair.md) and re-placed on a healthy disk on a different node.

The `shannonstore.data.storage.dirs` list is intentionally ordered: when two disks are equally desirable in the placement decision, the earlier-listed disk wins ties. This gives a stable, reproducible placement order across restarts.

## Same-partition deduplication

If two configured dirs happen to live on the **same** underlying filesystem (e.g. an operator misconfiguration that listed `/data/shannon-1` and `/data/shannon-2` where both are subdirectories of one big disk), the data node detects the duplicate at startup:

```
DEBUG DataConfig - Skipping redundant storage directory on same partition: /data/shannon-2
```

Only the first such entry is added to the active storage set. This prevents the placement algorithm from believing it has more failure domains than it does — a per-shard random pick across "two disks" that are really one would over-load a single mount.

## Disk health

Each data node maintains a small in-memory `DiskInfo` record per configured disk:

| Field | Source |
| --- | --- |
| `path` | the configured directory |
| `total / used / available` bytes | filesystem statvfs at the configured refresh interval |
| `available` (boolean) | last write or read succeeded |
| `lastSeenMs` | timestamp of the last successful access |
| `chunkCount` | bookkeeping from `ChunkStore` |

A scheduled probe refreshes the byte counts every `shannonstore.data.disk.info.refresh.interval.seconds` (default 60). The probe is filesystem `statvfs` — cheap, async, never blocks the dataplane. If a probe fails (filesystem unmounted, IO error), the disk is marked unavailable and excluded from new placements until two consecutive probes succeed again.

The data node publishes a summary heartbeat to the API tier:

```text
data node → API tier:
  {
    nodeId:   "data-12",
    disks: [
      { path: "/mnt/disk-1", total: 8 TiB, used: 6.4 TiB, available: true, chunkCount: 154 783 },
      { path: "/mnt/disk-2", total: 8 TiB, used: 6.5 TiB, available: true, chunkCount: 155 921 },
      { path: "/mnt/disk-3", total: 8 TiB, used: 0,        available: false, chunkCount: 0      },
      { path: "/mnt/disk-4", total: 8 TiB, used: 6.4 TiB, available: true, chunkCount: 154 503 }
    ]
  }
```

The API tier uses these heartbeats for placement decisions (HRW excludes unavailable disks), capacity reports in the Admin UI, and the trigger condition for the Disk Repair Service.

## Adding a disk

Mount it, edit `shannonstore.data.storage.dirs`, restart the data node. On restart:

1. The data node discovers the new directory in the configured list.
2. The first health probe records its capacity.
3. The heartbeat to the API tier advertises the new disk.
4. HRW placement starts considering it for new writes immediately.
5. **No data is migrated to fill the new disk** — placement converges naturally as new writes accumulate. Over time the new disk catches up with the others' utilization.

If the operator wants the new disk warm immediately, run a manual repair sweep (`POST /admin/disk-repair/start` on the API node) which redistributes shards more aggressively. This is the only path that actively moves *existing* data to a *new* disk.

## Removing a disk

The graceful sequence:

1. Mark the disk's mount as drained externally (read-only).
2. The data node's probe registers the disk as unavailable on the next refresh.
3. The Disk Repair Service picks up the shards from that disk's chunk store and re-places them on healthy disks across the cluster.
4. When the chunk count for that disk reaches zero, the disk is empty and the mount can be detached safely.

For a *failed* disk that's already gone, the steps are identical without the manual mark — the probe fails, the disk drops out of placement, repair fires. The only difference is that the disk's reads can't satisfy any in-flight requests during the repair, so the cluster temporarily reconstructs via parity for the shards that were on the failed disk.

## Operational considerations

- **Disks should be similar in size**. HRW placement is uniform across the available set, so a 16-TiB disk in a node otherwise full of 8-TiB disks will fill twice as fast and become the bottleneck.
- **Distinct filesystems matter**. The same-partition deduplication above is a safety net, not a recommendation; configure each disk as its own filesystem.
- **Watch for `available: false` in the heartbeat**. The Admin UI's data-node panel highlights any disk in this state. A single disk going unavailable for more than the repair grace window triggers reconstruction.
- **Plan for the rebuild window**. With k=4 m=2, losing a disk on a node holding ~150 000 shards rebuilds roughly proportional to the EC throughput per second. A throttled repair (default rate-limit is per-cluster, not per-disk) takes hours to days for a multi-terabyte disk — clusters that need shorter windows should raise `shannonstore.api.disk.repair.rate.limit.bytes.per.sec`.

## See also

- [Erasure Coding](ec.md) — the cluster-level redundancy that makes JBOD safe.
- [Disk Repair Service](disk-repair.md) — the worker that re-places shards from a failed disk.
- [Data Integrity](data-integrity.md) — the per-shard CRC32C that catches silent corruption on individual disks.
- [Distributed Architecture](distributed-architecture.md) — the cluster topology that hosts the data nodes.
- [Monitoring & Metrics](monitoring.md) — disk-level capacity metrics surfaced through Prometheus.
