# Backup &amp; Restore

ShannonStore can periodically ship the entire cluster's state — IAM, KMS, metadata indexes, and every data node's chunk store — to an external S3-compatible target on a schedule. The backup is incremental and opt-in.

The destination is any AWS-compatible S3 endpoint: another ShannonStore cluster, MinIO, AWS S3, or similar. Pointing it at a different cluster gives you a true offsite copy.

## What Gets Backed Up

A single backup run captures both halves of the cluster's state under one shared backup id, so any single restore reassembles a consistent point in time:

- **API server (leader)** — IAM RocksDB, KMS RocksDB, metadata / index RocksDB, metrics & history stores, and the bucket-config RocksDB (policies, CORS, lifecycle, tags, storage classes, bucket and site replication, notification targets). The bucket config is KMS-encrypted, so its copy in the backup holds no secret in clear; it opens with the KMS keystore of the same backup.
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

The same page also has a **Test Connection** button (probes the destination bucket with the values on the form, saved or not) and a **Backup now** button. **Backup now** starts the run on the leader and returns at once; the run shows as *In progress* and then in the history. Starting a second run while one is going answers `409`.

Saving refuses an invalid cron expression, and refuses to enable backups without an `http://` or `https://` endpoint, a bucket, an access key and a secret key — a schedule with no target would otherwise fail on every tick. The secret key is write-only: the page shows that one is stored and a blank field keeps it.

The scheduler and the run history live on the leader; every API node answers these pages with the leader's data. The settings are kept on **every** API node: the leader sends them to the others inside the encrypted IAM snapshot, so whichever node becomes leader next runs the scheduled backups and can restore. On disk they are `data/s3-metadata/backup-config.enc`, sealed with a key derived from `SHANNONSTORE_MASTER_KEY` (the file used to be plain `backup-config.json`; the first start of this release converts and deletes it, and a follower that still has settings from that time offers them to a leader that has none). The settings are not part of the backup, so a restore never rolls the destination back.

The request the leader sends to every data node at the start of a backup or restore carries the destination's secret key and is KMS-encrypted on the wire.

!!! warning "Use another cluster as the target"
    Backing up into a bucket of the same cluster works for a test, but each run then includes the previous backups (they are chunks on the same data nodes), so the backup grows with every run.

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

1. Operator clicks **Restore** on the chosen history row (`POST /admin/restore {"backupId": ...}`).
2. The leader API server starts staging in the background: it pulls every file in that backup's manifest into staging directories that sit *next to* the live RocksDB / chunk-store dirs (e.g. `iam.restore-pending/`).
3. The leader fans out the same request to every data node, which stages its own slice in parallel.
4. The admin UI polls `GET /admin/restore/status` (`inProgress`, `last`) and, when staging is done, shows the per-node report and asks the operator to **restart the cluster**.
5. On the next process start, each component checks for staged dirs, archives the live one to `*.restore-old-<ts>/`, and atomically renames the staged dir into place.

The two-stage design (stage now, swap on restart) avoids hot-swapping directories that RocksDB and the chunk store hold open file handles on. Old snapshots remain on disk as `*.restore-old-<ts>` until the operator removes them, so a restore is reversible by hand.

Restore is leader-driven and writes through the same fan-out as backup; followers can read but not initiate.

A directory the chosen backup has no files for — a bucket-config store when restoring a backup taken by a release that did not back it up yet — is left as it is: it gets no staging directory, so the restart does not replace the live directory with an empty one.

## Limitations

- The API-side slice of a backup is stored under the id of the API node that was leader when it ran (`api-<node id>/`). A restore reads the slice of the API node that runs it, so restore a backup on the API node that took it (make it the leader first), or take a new backup on the current leader.
- A node whose `advertisedHost` has changed since the backup will not find its manifest. Bring up the replacement node with the original host name first, or re-take a baseline backup before mutating cluster identity.
- The cluster restart in step 5 is operator-driven on purpose — the swap is a coordinated cluster-wide action, not a per-node one.
