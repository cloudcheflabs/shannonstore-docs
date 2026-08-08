# Maintenance Mode

Maintenance mode is a single cluster-wide toggle that tells every API node "shed S3 client load now". When active, the data plane on port `:8080` responds to every S3 request with `503 Service Unavailable` and `Retry-After: 30`; well-behaved SDKs back off and the operator gets a stable, in-flight-only window to perform disruptive work. The admin plane on `:8888` is **not** affected — operator tooling, monitoring, and cross-product credential sweepers keep working.

```text
   Client     ┌──────────────────────────────────┐     Admin
   ──────────►│  API node (maintenance = true)   │◄──────────
   :8080      │                                  │     :8888
              │   if (maintenanceMode):           │
              │     return 503 + Retry-After:30  │
              │   else:                          │
              │     dispatch normally            │
              └──────────────────────────────────┘
```

`S3RequestHandler` checks maintenance mode early, but not before every other gate: the cluster-ready check (self/cluster not yet initialized → `503` with no `Retry-After`) and the CORS-preflight / `Authorization`-header-presence gate (a request with no header and no matching anonymous bucket policy still gets `401 Unauthorized`) both run first. Maintenance mode is checked immediately after that — before SigV4/SigV2 signature verification, access-key validation, or dispatch — so a request that *does* carry some form of authorization is turned away with `503` before its signature is ever checked. The mode is a single in-memory `AtomicBoolean` flipped via the admin REST and broadcast to every peer API node, so a `true` value on the leader is visible everywhere within the IAM/bucket sync round-trip.

## When to use

| Use it for | Don't use it for |
| --- | --- |
| Replacing a failed disk on a data node | Adding a new data node (HRW rebalances automatically) |
| Draining in-flight uploads before evacuating a node | Adding a new API node (membership is hot) |
| Cluster-wide KMS rotation steps that require a quiescent dataplane | Single-node restarts (peers absorb the load) |
| Backup-restore that overwrites IAM/bucket state | Routine config tuning that doesn't change addressing |
| Investigating data corruption without races against new writes | Anything that completes in under a minute (the SDK back-off would mask it) |

The right mental model: maintenance mode is *defense* — pause the cluster precisely when you don't want a write landing midway through your action. It is **not** a step in any routine scaling or upgrade procedure.

## Activation

`POST /admin/maintenance/mode` against any API node:

```bash
TOKEN=$(curl -sf -X POST http://localhost:8888/admin/auth/login \
    -H 'Content-Type: application/json' \
    -d '{"userId":"admin","password":"…"}' \
    | python3 -c "import sys,json;print(json.load(sys.stdin)['token'])")

# Enable
curl -sf -X POST http://localhost:8888/admin/maintenance/mode \
    -H "Authorization: Bearer $TOKEN" \
    -H 'Content-Type: application/json' \
    -d '{"enabled":true}'

# Disable
curl -sf -X POST http://localhost:8888/admin/maintenance/mode \
    -H "Authorization: Bearer $TOKEN" \
    -d '{"enabled":false}'
```

The same toggle is exposed in the Admin UI's cluster overview. Both paths go through `storageService.broadcastMaintenanceMode(enabled)`, which sets the local flag and pushes a `MaintenanceModeChanged` message over the internal NIO channel to every peer API node. Followers flip their own `AtomicBoolean` on receipt — no leader/follower divergence window.

## What clients see

While maintenance is active:

```
HTTP/1.1 503 Service Unavailable
Retry-After: 30
Content-Length: 0
```

AWS SDKs interpret `503 + Retry-After` as a back-off signal and reschedule with the duration the server suggested. With default exponential-backoff settings the retry budget is multi-minute, so a 5-minute maintenance window is invisible to the application other than a flat-line in throughput.

The handler returns 503 *before* any auth validation runs. This is intentional: signature validation against a partially-quiescent IAM state during sync would be racy, and there's nothing useful to do for a request the cluster has already decided to defer.

## What the dataplane is doing during the window

- New PUT / GET / DELETE / multipart requests bounce with 503 immediately.
- In-flight writes that started *before* the toggle continue to completion — the toggle is checked once per request at dispatch, not per chunk. Operators should briefly wait for the dataplane to quiesce before taking the disruptive action; the typical drain is well under a minute for an idle multi-gigabyte tail.
- The Bitrot Scrubber pauses (it checks `isMaintenanceMode()` between batches).
- The Disk Repair Service pauses for the same reason.
- Background metadata reconciliation pauses.
- The admin REST and Admin UI remain fully responsive.

Nothing on the admin port respects the flag — that's the whole point. The operator needs a working IAM, a working cluster status panel, and a working access-key minter while staring at a quiet dataplane.

## What the dataplane is **not** doing

Maintenance mode does not:

- **Stop accepting peer membership changes**. A data node joining or leaving during maintenance is handled normally; HRW reshuffles the affected shards.
- **Force connection drains**. Existing keep-alive TCP connections stay open. The 503 is the answer to the next request on each connection; new connections still get the same answer.
- **Touch the admin / port 8888 listener**. Verified on every release.
- **Persist across restarts**. The `AtomicBoolean` is in-memory only — if every API node in the cluster is restarted while in maintenance, the cluster comes back live. Use this carefully: a planned shutdown that intends to come back in maintenance must re-issue the POST immediately after start.

## Cluster-wide vs node-local

There is intentionally only one form: cluster-wide. A per-node "drain me out" doesn't exist, because the API tier rebalances around an unhealthy or absent node automatically without any operator signaling.

If an operator wants to evacuate a single API node, the right sequence is:

1. **Activate maintenance mode** (cluster-wide) so the data plane briefly pauses.
2. **Stop the target API node** — peers absorb its keys through the standard membership-change path.
3. **Deactivate maintenance mode**. Clients resume against the smaller cluster.
4. **Replace / repair / upgrade** the evacuated node out of band.
5. **Restart the node**; HRW migrates ownership of the keys it should now hold.

The brief maintenance window in step 1 is purely cosmetic — it makes the transition from N to N-1 nodes a *deliberate* dataplane event rather than a flap of 503s from the single absent node's leftover keys.

## Coordination with other services

| Service | Behaviour while maintenance is on |
| --- | --- |
| S3 dispatch (`S3RequestHandler`) | every request → 503 |
| Bitrot Scrubber | pauses between batches |
| Disk Repair Service | pauses |
| Metadata reconciliation sweep | pauses |
| Backup / Restore | continues — explicit operator action |
| Admin REST + Admin UI | continues |
| IAM / bucket-state cluster sync | continues |
| Prometheus `/metrics` endpoint | continues |

Pausing the background workers is a soft pause — they finish the current batch, then sit out until the flag clears. The pause is not enforced through hard cancellation, so a long-running EC reconstruction in flight when the toggle flips will complete.

## Diagnostics

The toggle's current state is included in the cluster-status JSON returned by `GET /admin/api/cluster/status`:

```json
{
  "leaderId":          "…",
  "apiNodes":          [ … ],
  "dataNodes":         [ … ],
  "maintenanceMode":   true,
  "serverReady":       true,
  "clusterReady":      true,
  "timestamp":         1779999999999
}
```

The Admin UI's overview header shows the same value. Log lines on every API node record activation and deactivation:

```
INFO  StorageService - Maintenance mode set to TRUE by admin
INFO  StorageService - Maintenance mode set to FALSE by admin
```

If a 503 is observed in production and the operator did not flip the toggle, two other paths produce the same status code — both also surface in logs:

- `serverReady == false` (cluster has not finished bootstrapping yet) — same 503, no `Retry-After`.
- Prometheus collection failure path returns 503 from `/metrics` only.

The dataplane's 503 + `Retry-After` is unambiguous: it means maintenance mode.

## See also

- [Cluster Operations](../operations/operations.md) — the operational runbook that uses maintenance mode in disruptive procedures.
- [Disk Repair Service](disk-repair.md) — one of the workers that pauses while the flag is on.
- [Data Integrity](data-integrity.md) — the Bitrot Scrubber lifecycle that respects the same flag.
- [Authentication & Authorization](auth-authz.md) — the admin token flow that's still required to flip the toggle.
