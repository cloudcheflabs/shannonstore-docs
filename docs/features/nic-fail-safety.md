# NIC Silent-Fail Safety

ShannonStore's internal NIO plane survives the trickiest failure mode in distributed storage: **a peer whose NIC dies after the socket was opened, but ZooKeeper hasn't expired the session yet**. From ZooKeeper's POV the node is alive. From every neighbour's POV its connection is also alive — TCP does not surface a half-open peer for tens of seconds, sometimes minutes, while the kernel exhausts its retransmit budget. Reads that route to that peer hang. Writes targeting it hang. ZooKeeper-driven failover doesn't fire because membership is still healthy.

The 1.0.0 release closes this gap on **both** ends of every internal RPC.

## What changed in the NIO client

`NioClient` (used by `StorageService` for every chunk PUT, every chunk GET, every metadata RPC, and every internal pull) now:

- **Bounded `connect()`** — the channel is opened in non-blocking mode, a per-call `Selector` waits on `OP_CONNECT` with a 10s deadline. A peer mid-3-way-handshake whose NIC went dark used to park the caller for the OS retransmit budget; now it raises `SocketTimeoutException` after 10s.
- **Bounded `send()`** — same pattern on `OP_WRITE`. A plain blocking `channel.write()` returns 0 the instant the OS send buffer fills up and the peer stops draining. The old loop had no deadline; it spun forever. The new loop wakes only on writability or the deadline.
- **`Selector`-based receive (existing behaviour, preserved)** — read timeout was already bounded.

Both defaults are 10s, overridable per JVM with:

```text
-Dshannonstore.network.nio.client.connect.timeout.seconds=10
-Dshannonstore.network.nio.client.write.timeout.seconds=10
```

Wired through `ApiConfig` keys:

```properties
shannonstore.network.nio.client.connect.timeout.seconds = 10
shannonstore.network.nio.client.write.timeout.seconds   = 10
```

## What changed in the connection pool

Before the fix, a failed RPC's connection was always returned to the pool. `ConnectionPool.returnConnection()` only closes the socket when `client.isConnected()` reads false — but `SocketChannel.isConnected()` is `true` for the entire OS retransmit window even when the peer is silently dead. The next caller would borrow the half-open socket and hang again on its own write.

After the fix, both PUT (`StorageService.putAndCommitChunk`) and GET (`StorageService.fetchFromDataNodes`) call `invalidateConnection()` on the failure path. That close + permit-release lets the next caller mint a fresh socket instead of inheriting the wedge.

## What changed in the write path — no more silent reduced redundancy

A separate gap, addressed in the same release: **write-time parity loss used to vanish from metadata**.

EC encode produces k+m shards. The PUT path fans them out to k+m disks in parallel and commits as soon as the k data shards ACK. If any of the m parity PUTs failed mid-flight (a NIC silent fail being the canonical case), the failed shard was *dropped on the floor*. The object's metadata recorded only the k+m−1 (or fewer) shards that succeeded. The periodic disk-repair scrubber walked metadata, never saw the missing slot, and the object lived out its life at reduced redundancy. No metric flagged it.

The 1.0.0 release fixes both halves of the gap:

1. Every shard slot **lands in metadata** even when its PUT failed. The slot is filled as MISSING — empty `nodeAddress`, zero checksum. Readers see it and skip it; EC parity reconstructs the data; the slot remains as a visible "this shard is owed" marker.
2. `storeObject` and `processBufferedPart` hand the set of missing shard indices to a new **reactive repair queue** on `DiskRepairService`. A daemon worker drains the queue every 200ms, rebuilds each missing shard from the surviving k via EC parity, writes the reconstructed bytes to a fresh disk picked by the placement strategy, and updates the `ChunkInfo` to point at the new home. Leader-only so the metadata write stays on a single coordinator; non-leaders re-queue with back-off.

The reactive worker exposes four new counters via `GET /admin/maintenance/repair/status`:

```json
{
  "reactiveQueueSize":    0,
  "reactiveSubmitted":  142,
  "reactiveSucceeded":  140,
  "reactiveFailed":       2
}
```

Operators see write-time partial failures as soon as they happen, and watch the queue drain back to zero as redundancy is restored.

## Test artifacts

- `tests/test-nic-failure-pr14.sh` — pauses one data-node mid-PUT, asserts the PUT returns within the configured timeout window instead of blocking on the kernel retransmit budget.
- `tests/test-reactive-repair-pr15.sh` — pauses one data-node, drives a PUT through the buffered path, watches the `reactiveSubmitted` / `reactiveSucceeded` counters tick, and verifies the resulting object round-trips byte-for-byte once the reactive worker has restored redundancy.

Both run against the standard `tests/docker-compose-shannonstore.yml` stack.

## When this fires in practice

| Scenario | Old behaviour | 1.0.0 behaviour |
|---|---|---|
| Data-node NIC silent fail mid-read | GET parked on dead peer for OS retransmit budget (minutes) | `Selector` deadline raises after 10s, EC parity reconstructs from k surviving shards, GET returns within fetch-timeout |
| Data-node NIC silent fail mid-write (parity shard) | k data shards ACK → metadata commits with parity slot dropped → permanent reduced redundancy | k data shards ACK → metadata commits with parity slot MISSING → reactive worker rebuilds within seconds |
| Data-node NIC silent fail mid-write (data shard) | Old write hung waiting for the k-quorum latch | `Selector` deadline raises, EC's remaining shards (other data + parity) finish the k-quorum from healthy peers; failed slot enters MISSING + reactive repair |
| Connection pool entry to silently-failed peer | Reused on next request, hung again | `invalidateConnection()` closes + releases permit; next mint is fresh |
