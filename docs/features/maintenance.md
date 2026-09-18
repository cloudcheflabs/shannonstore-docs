# Maintenance Mode

Maintenance mode is a cluster-wide switch that stops ShannonStore serving client traffic, so an operator can replace a disk, rebalance, or restore metadata without writes landing in the middle of the work.

While it is on, the S3 surface on `:8080` answers every request with `503 Service Unavailable` and a `Retry-After` header, and the admin console's **data-changing** routes answer the same. Reads, settings, and the maintenance switch itself stay open — the operator has to be able to watch progress and turn the window off again.

Background work — disk repair, the bitrot scrubber, rebalance, lifecycle expiry — **keeps running**. That is the point of the window, not something it suspends.

```text
   S3 client   ┌───────────────────────────────────────┐   Admin console
   ───────────►│  API node  (maintenance = true)       │◄──────────────
   :8080       │                                       │   :8888
               │   S3 request          → 503 Retry-After
               │   admin data write    → 503 Retry-After
               │   admin read/settings → served        │
               │   background workers  → still running │
               └───────────────────────────────────────┘
```

## Where the switch is checked

On the S3 path it is the **first** gate after the cluster-ready check — before CORS preflight, before the `Authorization` header check, before anonymous bucket-policy evaluation, and before any signature verification.

That ordering matters. It used to sit below the auth gate, which meant an anonymous read of a public bucket was served by the anonymous path before the check was ever reached: a repair would be racing reads it believed had stopped. Answering `503` to a caller holding no credentials leaks nothing — a service being unavailable is observable either way.

So while maintenance is on, **every** S3 request gets `503`, authenticated or not.

On the admin path the check runs after authentication and applies only to routes that change stored data:

| Blocked | Open |
| --- | --- |
| `POST` / `DELETE /admin/browser/buckets` | every `GET` |
| `DELETE /admin/browser/objects/…` | `/admin/maintenance/*` |
| `PUT /admin/browser/bucket-config/…` | `/admin/auth/*` |
| `POST /admin/browser/upload` | IAM, KMS, storage-class, lifecycle and other settings |

Settings edits stay open deliberately: changing a lifecycle or storage-class policy during a maintenance window is a normal thing to be doing, and blocking it would stop the very work the window was opened for.

## Where the setting lives

In the cluster config store — RocksDB, alongside the other cluster-global settings such as the site-replication config — and it travels to peer API nodes in the config snapshot, the same channel that carries IAM and bucket state.

ZooKeeper would have been the other candidate and is the wrong one: ZooKeeper holds **node state** (membership, leadership, readiness) while **settings** live in RocksDB and replicate through the snapshot. That is the split across every Cloud Chef Labs product.

Three consequences follow, and all three are why it is stored rather than held in memory:

- **It survives an API node restart.** A node restarted during a window reloads the setting from its own store on boot and comes back still refusing traffic.
- **It survives a full cluster restart.** Every node reloads independently.
- **A node that joins mid-window picks it up** from the peer snapshot rather than serving traffic because it happened to boot.

Each node also keeps the value in memory as a read cache, because every S3 request consults it and a per-request store read would put the setting in the hot path. The store is the source of truth; the cache is what the request path reads.

## When to use it

| Use it for | Don't use it for |
| --- | --- |
| Replacing a failed disk on a data node | Adding a data node (HRW rebalances on its own) |
| Draining in-flight uploads before evacuating a node | Adding an API node (membership is hot) |
| KMS rotation steps that need a quiet data plane | Single-node restarts (peers absorb the load) |
| Restoring metadata over live IAM / bucket state | Routine config tuning that does not change addressing |
| Investigating corruption without racing new writes | Anything finishing inside a minute — SDK back-off hides it anyway |

The mental model is *defence*: close the cluster exactly when a write landing midway through your action would be a problem. It is not a step in routine scaling or upgrades.

## Turning it on and off

From the Admin UI, **Cluster Nodes → Enter Maintenance**. A confirmation appears first — it stops S3 for the whole cluster — and a banner stays on screen while the window is open. The banner is amber while the setting is still reaching every node and red once every node reports it, so an operator does not start work during the propagation gap.

Or over REST, against any API node:

```bash
TOKEN=$(curl -sf -X POST http://localhost:8888/admin/auth/login \
    -H 'Content-Type: application/json' \
    -d '{"userId":"admin","password":"…"}' \
    | python3 -c "import sys,json;print(json.load(sys.stdin)['accessToken'])")

# on
curl -sf -X POST http://localhost:8888/admin/maintenance/mode \
    -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
    -d '{"enabled":true}'

# off
curl -sf -X POST http://localhost:8888/admin/maintenance/mode \
    -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
    -d '{"enabled":false}'
```

Both paths call `broadcastMaintenanceMode()`, which writes the setting to the local store and then pushes `TYPE_MAINTENANCE_MODE_REQ` over the internal NIO channel to every peer. Peers write it to their own store on receipt. The broadcast is a latency optimisation — it makes the change visible immediately rather than at the next snapshot sync — and the store is what makes it stick.

!!! note "Wait for every node before starting work"
    The switch is cluster-wide but applied per node. There is a brief window where one node has it and another does not. Confirm every node agrees before pulling a disk:

    ```bash
    curl -sf "http://localhost:8888/admin/maintenance/status?targetHost=<node>&targetPort=8888" \
        -H "Authorization: Bearer $TOKEN"
    ```

    The Admin UI does this for you — that is what the amber banner means.

## What clients see

```
HTTP/1.1 503 Service Unavailable
Retry-After: 30
Content-Length: 0
```

AWS SDKs read `503` plus `Retry-After` as a back-off signal and reschedule after the interval the server asked for. With default exponential back-off the retry budget runs to several minutes, so a short window shows up in an application as a flat spot in throughput rather than as errors.

`Retry-After` is configurable:

```properties
# How long a client is told to wait before retrying while the cluster is in
# maintenance mode (seconds). Set it to roughly how long the window usually
# lasts: too low and clients hammer a cluster that is deliberately closed,
# too high and they stay away well after it reopens.
shannonstore.api.maintenance.retry.after.seconds=30
```

The same value is used for the S3 surface and the admin routes, so a client sees one behaviour whichever door it knocked on.

## What happens during the window

- New PUT / GET / DELETE / multipart requests are refused immediately.
- Requests that were **already in flight** when the switch flipped run to completion — the check happens once per request at dispatch, not per chunk. Wait for the data plane to quiesce before taking the disruptive action; for an idle cluster that is well under a minute.
- **Background workers keep running.** Disk repair, the bitrot scrubber, rebalance, lifecycle expiry, metadata reconciliation and replication all continue. None of them consults the maintenance switch, by design: they are usually the work the window exists to protect.
- Admin reads, settings changes, IAM, KMS and `/metrics` continue.

### What it does not do

- **It does not drain connections.** Existing keep-alive TCP connections stay open; the `503` is simply the answer to the next request on each.
- **It does not stop membership changes.** A data node joining or leaving during the window is handled normally and HRW reshuffles the affected shards.
- **It does not pause background workers** — see above. To stop a worker, stop that worker: the scrubber, lifecycle scanner, reconciler and replication each have their own enable toggle under `/admin/maintenance/…/enable`.
- **It does not block settings changes** through the admin console.

## Evacuating a single API node

There is only one form of the switch: cluster-wide. There is no per-node drain, because the API tier routes around an absent node on its own.

To take one API node out:

1. **Turn maintenance on** and wait for every node to report it.
2. **Stop the target node** — peers pick up its keys through the normal membership path.
3. **Turn maintenance off.** Clients resume against the smaller cluster.
4. **Repair or upgrade** the node out of band.
5. **Start it again**; HRW migrates ownership of the keys it should now hold.

The window in step 1 makes the N → N−1 transition one deliberate pause rather than a scatter of failures from the departing node's leftover keys.

## Checking the state

```bash
curl -sf http://localhost:8888/admin/maintenance/status \
    -H "Authorization: Bearer $TOKEN"
```

```json
{
  "maintenanceMode": true,
  "…": "background task statuses"
}
```

Add `?targetHost=<host>&targetPort=<port>` to ask one specific node rather than whichever one answered — that is how you confirm the setting reached the whole cluster.

Every node logs the transition:

```
INFO  BucketManager  - Cluster maintenance mode set to: true
INFO  StorageService - Maintenance mode set to: true
```

Two lines because two things happened: the setting was written to the store, then the read cache was updated. A node that restores the setting on boot logs it differently:

```
INFO  StorageService - Maintenance mode restored from persisted config: true
```

Seeing that line after a restart is the confirmation that the window held.

### A 503 you did not ask for

Two other paths answer `503` on the data plane:

- **Not ready yet** — `serverReady` or the cluster-ready flag is false while a node bootstraps or a leader transition completes. Distinguishable by its shorter `Retry-After`.
- **A reverse proxy with no live upstream.** nginx answers `502` rather than `503`, but it is worth knowing about when diagnosing a cluster under load.

If the response carries the configured maintenance `Retry-After`, it is maintenance mode. Check `/admin/maintenance/status` to be sure.

## Verifying it

`tests/test-maintenance-mode-e2e.sh` in the product repository exercises the whole cycle on a compose cluster: a signed S3 round-trip before the window, refusal during it — including the admin routes — **an API node restarted mid-window still refusing**, a signed round-trip again afterwards, and the object written before the window reading back byte-for-byte.

The restart is the part that matters. A test without it passes just as happily against an implementation that keeps the switch only in memory.

## See also

- [Cluster Operations](../operations/operations.md) — the runbook that uses maintenance mode in disruptive procedures.
- [Disk Repair Service](disk-repair.md) — a worker that keeps running during a window.
- [Data Integrity](data-integrity.md) — the bitrot scrubber, which has its own enable toggle.
- [Authentication & Authorization](auth-authz.md) — the admin token flow needed to flip the switch.
