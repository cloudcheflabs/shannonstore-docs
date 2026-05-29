# Disk Repair Service

`DiskRepairService` is the worker that turns disk loss back into full redundancy. A disk dies; its shards are now under-replicated; the service finds them, reconstructs the missing bytes from erasure-coded parity, and places the rebuilt shards on healthy disks elsewhere in the cluster. The same code path serves silent-corruption recovery from the [Bitrot Scrubber](data-integrity.md) — corrupt shards are treated identically to missing ones.

```text
   ┌───────────────────────────────────────────────────────────┐
   │  Reactive trigger                                          │
   │   GET path found a missing or CRC-mismatched shard         │
   │   → enqueue repair task for that shard                     │
   │                                                            │
   │  Proactive triggers                                        │
   │   Bitrot Scrubber found a CRC mismatch                     │
   │   Disk-health probe marked a disk unavailable              │
   │   Operator hit "Repair now" in the Admin UI                │
   └────────────────┬───────────────────────────────────────────┘
                    │
                    ▼
        ┌───────────────────────┐
        │  Repair task queue     │
        └────────┬───────────────┘
                 │  (bounded worker pool)
                 ▼
   ┌───────────────────────────────────────────────────────────┐
   │  For each task:                                            │
   │   1. Look up the object's metadata                         │
   │   2. Fetch k healthy shards (data or parity) from peers    │
   │   3. Reconstruct the missing shard via Reed-Solomon decode │
   │   4. Pick a target disk on a different node                │
   │   5. Write the rebuilt shard to the target                 │
   │   6. Update the metadata's shard layout entry              │
   │   7. Replicate the metadata to peer API nodes              │
   └───────────────────────────────────────────────────────────┘
```

The repair is reactive *and* proactive on purpose: the GET path doesn't wait for a scheduled scan to learn that a shard is gone, and the scheduled scan doesn't wait for a GET to drive coverage of cold data. The two paths feed the same queue and the same worker pool.

## What triggers a repair

| Trigger | Initiator | Latency to repair |
| --- | --- | --- |
| GET found shard missing | `S3RequestHandler` (read path) | seconds — task enqueued before the GET response goes back |
| GET found CRC mismatch | `S3RequestHandler` | seconds |
| Bitrot Scrubber CRC mismatch | `BitrotScrubberService` | minutes — the scrubber batches before emitting |
| Disk-health probe failure (sustained) | data-node heartbeat | minutes after grace window |
| Manual trigger | operator via Admin UI / `POST /admin/disk-repair/start` | immediate |

The grace window on disk-health is essential — a transient mount glitch (filesystem freezing while flushing, kernel-level retry) should not kick off a multi-hour rebuild of a disk that's about to come back. Default grace is one minute of sustained unavailability; tunable via configuration.

## The repair pipeline

For each task the worker:

1. **Loads the object metadata** for the affected `(bucket, key, version)` from the local index. Repair operates on one *shard slot* of one *part* at a time — for a multipart object only the affected parts get reconstructed.
2. **Identifies k healthy shards** from the part's layout. The same Reed-Solomon decoder used by client GET (see [Erasure Coding](ec.md)) accepts any k of the k+m, so the worker prefers data shards first and falls back to parity shards as needed.
3. **Fetches the k shards** from their owning data nodes through the pooled NIO channel. Failures during the fetch retry against alternate sources of the same shard slot if available, then enlarge the source set to include another parity shard.
4. **Reconstructs the missing shard** locally using the inverse Vandermonde matrix multiply over GF(2⁸).
5. **Picks a target disk** for the rebuilt shard. Constraint: the target must be on a different data node than the lost shard's owner *and* on a healthy disk *and* — when failure-domain awareness is in use — in a different domain than every other shard of the same `(key, part)`. HRW provides the placement deterministically given the constraint set.
6. **Writes the rebuilt shard** to the target disk and persists the chunk file under `data/<storage-dir>/`. The data node returns the `chunkId` and CRC32C of what it just wrote.
7. **Updates the metadata** for the part to point the shard slot at the new `{nodeAddress, diskPath, chunkId, checksum}`.
8. **Replicates the metadata** through the standard replication channel so peer API nodes see the new placement on their next lookup.

The single-task surface above is what gets parallelized — multiple workers process distinct tasks concurrently subject to the configured concurrency cap.

## Throttling

A naive rebuild of a 4 TiB disk would saturate every other disk and every NIC in the cluster for hours. The service throttles in three dimensions:

| Knob | Default | Effect |
| --- | --- | --- |
| `shannonstore.api.disk.repair.scan.interval.seconds` | minutes | How often the proactive sweep checks for under-replicated shards. |
| `shannonstore.api.disk.repair.concurrency` | small (4) | Number of parallel repair workers. |
| `shannonstore.api.disk.repair.rate.limit.bytes.per.sec` | configured | Cluster-wide cap on rebuilt-shard write throughput. |
| `shannonstore.api.disk.repair.min.target.disk.bytes` | configured | Minimum free bytes a target disk must have to accept a rebuilt shard. |

The rate limit is the single most operationally important knob. On a cluster where client throughput is the priority and a few hours of partial under-replication is acceptable, throttle low (10–50 MiB/s). On a cluster where every minute under-replicated is a compliance risk, throttle high (hundreds of MB/s) and accept the dataplane contention.

Repairs **pause** when maintenance mode is active — the operator likely flipped maintenance precisely to perform the kind of work that would interfere with repair.

## Grace period

A disk that fails one probe is not immediately written off. The grace period gives a transient fault time to clear:

| State | Behaviour |
| --- | --- |
| Probe failed once | mark unavailable; new placements skip the disk; existing reads of its shards fall back to parity |
| Probe failed for grace window | enqueue repair tasks for shards on the disk |
| Probe recovered before grace expired | reinstate the disk; cancel any pending tasks for shards on it |

The grace window is configurable, default one minute. It does **not** protect against repeated short failures — five consecutive 10-second flaps still trigger repair because the disk is clearly unstable.

## Manual trigger

Operators can force a repair sweep through the Admin REST or Admin UI:

```bash
curl -sf -X POST http://localhost:8888/admin/disk-repair/start \
    -H "Authorization: Bearer $TOKEN"
```

The sweep walks every object's layout and emits a task for any shard whose owning disk is currently unavailable. This is the right path for:

- An operator who just added a new data node and wants to warm-start it with existing shards (the sweep migrates a fraction of cold data to the new node).
- Cluster-wide reassurance after a long maintenance event.
- One-off recovery after disabling and re-enabling the scrubber.

Status comes back from `GET /admin/disk-repair/status`:

```json
{
  "enabled":             true,
  "scanInProgress":      false,
  "lastCycleStartedAt":  1779999000000,
  "lastCycleFinishedAt": 1779999900000,
  "totalRepaired":       12483,
  "totalRepairFailed":   2,
  "queuedTasks":         0
}
```

## Failure modes

A repair task can fail for two reasons:

1. **Insufficient source shards** — fewer than k healthy shards exist anywhere in the cluster. The repair is impossible until at least one more shard becomes available; the task re-queues with a backoff and a critical log line is emitted. Operators should investigate immediately — this is the data-loss precursor.
2. **No eligible target disk** — every healthy disk is too full to accept the rebuilt shard, or the placement constraints (failure-domain, distinct-node) admit no candidate. The task re-queues; operators should add capacity or relax constraints.

Both failures increment `shannonstore.disk.repair.repairs.failed.total`, which the recommended alert rule (see [Monitoring & Metrics](monitoring.md)) catches as `increase(... > 0)` over five minutes.

## Coordination with the Bitrot Scrubber

The scrubber doesn't repair shards directly — it computes the CRC32C of each shard it reads and, on mismatch, enqueues a repair task with the same shape as a GET-found mismatch. The repair service then handles it identically to a missing shard: pull k healthy shards, reconstruct the corrupt slot from parity, write the rebuilt shard, update metadata. From the operator's perspective there's exactly one repair pipeline and exactly one set of repair counters; the trigger (read-path, scrubber, manual) is just a label on the source.

## See also

- [Erasure Coding](ec.md) — the parity scheme that makes repair possible.
- [Data Integrity](data-integrity.md) — the CRC32C and scrubber that catches silent corruption.
- [Multi-Disk Storage](multi-disk-storage.md) — the JBOD layout repair fans out across.
- [Maintenance Mode](maintenance.md) — pauses repair workers during disruptive operations.
- [Monitoring & Metrics](monitoring.md) — the counters every repair updates.
- [Distributed Architecture](distributed-architecture.md) — the cluster topology repair coordinates over.
