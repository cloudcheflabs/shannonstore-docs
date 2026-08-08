# Backup &amp; Restore

ShannonStore can periodically ship the entire cluster's state — IAM, KMS, metadata indexes, and every data node's chunk store — to an external S3-compatible target on a schedule. The backup is incremental and opt-in.

The destination is any AWS-compatible S3 endpoint: another ShannonStore cluster, MinIO, AWS S3, or similar. Pointing it at a different cluster gives you a true offsite copy.

## What Gets Backed Up

A single backup run captures both halves of the cluster's state under one shared backup id, so any single restore reassembles a consistent point in time:

- **API server (leader)** — IAM RocksDB, KMS RocksDB, metadata / index RocksDB, metrics & history stores.
- **Every data node** — the raw chunk-store directories (one per configured storage disk).

Each node uploads its own slice directly to the external S3, so the leader doesn't become a bottleneck and ShannonStore's internal S3 API isn't on the backup path.

## Configuration

The `S3 Backup` page in the admin UI exposes the full configuration:

- **Enabled** — master switch (default: off).
- **S3 endpoint, region, bucket, prefix** — the destination.
- **Access key / secret key** — long-lived static credentials for the destination.
- **Path-style addressing** — required for ShannonStore / MinIO destinations.
- **Interval (minutes)** — how often the scheduled backup runs.
- **Cron** — optional 5-field UNIX cron expression for wall-clock schedules (e.g. `0 2 * * *` for daily at 02:00). Coexists with the interval rule — both can be set, whichever comes due first fires the backup.
- **Retention (days)** — how long the visible history is kept (default 30).

The same page also has a **Test Connection** button (probes the destination bucket) and a **Backup now** button (triggers an immediate backup).

### Cron vs. Interval

The two automatic rules are independent and share the same tick loop:

- **Interval** is a relative "at least N minutes since the last successful backup" timer. Use it for safety-net cadence ("never let more than 6 hours pass between backups").
- **Cron** is wall-clock based. Use it for predictable schedules ("nightly at 02:00", "every Sunday at 06:00").

The cron expression is evaluated in the leader's local time zone. The first sighting after the cron is set, or after a leader handoff, **arms from "now"** — ShannonStore does not back-fire missed cron times on startup. An invalid cron is rejected at config-save time (the response body carries `{"status":"error", ...}`, HTTP 200) rather than being persisted and silently skipped at tick time.

## Incremental by Design

Files are content-addressed: anything whose `(path, size, mtime)` hasn't changed is matched against the destination and skipped. The cost in steady state is one HEAD per file plus uploads of just the parts that actually changed. A re-run with no cluster activity uploads zero bytes.

## History Retention

Past runs show up in the admin UI and age out by the **Retention (days)** policy. When an entry passes retention it disappears from the visible list; the underlying objects stay in S3 unless an S3 lifecycle rule removes them. This keeps storage cleanup explicit — operators decide when to actually reclaim space.

## Compared to Object-Level Tools

`aws s3 sync` from one ShannonStore to another also works as a *data* export — it copies the user-visible objects through ShannonStore's S3 API. This built-in backup is a *cluster-state* copy: it includes the metadata, IAM, and KMS state without which a restored cluster wouldn't have any users, keys, or buckets at all. It also runs in parallel from each node directly to the destination, without re-assembling chunks through the S3 protocol on the way out.

The two are complementary — pick "object sync" for data export, "S3 Backup" for disaster recovery.

## Restore

The same `S3 Backup` admin page lists the backup history. Each successful entry has a **Restore** button that points the cluster back at that snapshot.

The flow is:

1. Operator clicks **Restore** on the chosen history row.
2. The leader API server pulls every file in that backup's manifest into staging directories that sit *next to* the live RocksDB / chunk-store dirs (e.g. `iam.restore-pending/`).
3. The leader fans out the same request to every data node, which stages its own slice in parallel.
4. The admin UI reports per-node staging results and asks the operator to **restart the cluster**.
5. On the next process start, each component checks for staged dirs, archives the live one to `*.restore-old-<ts>/`, and atomically renames the staged dir into place.

The two-stage design (stage now, swap on restart) avoids hot-swapping directories that RocksDB and the chunk store hold open file handles on. Old snapshots remain on disk as `*.restore-old-<ts>` until the operator removes them, so a restore is reversible by hand.

Restore is leader-driven and writes through the same fan-out as backup; followers can read but not initiate.

## Limitations

- A node whose `advertisedHost` has changed since the backup will not find its manifest. Bring up the replacement node with the original host name first, or re-take a baseline backup before mutating cluster identity.
- The cluster restart in step 5 is operator-driven on purpose — the swap is a coordinated cluster-wide action, not a per-node one.
