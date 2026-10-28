# Admin REST API

Everything the [Admin UI](admin-ui.md) does goes through this REST API on the API server's admin port
(default `8888`, or `/admin` behind nginx). Any API server accepts any call.

## Conventions

- **Authentication.** `POST /admin/auth/login` with `{"userId": "...", "password": "..."}` (or
  `username`) returns `{"token", "refreshToken", "isAdmin", "requirePasswordChange", ...}`. Send the token
  as `Authorization: Bearer <token>`. `POST /admin/iam/keys` also returns a long-lived `token` that works
  the same way for automation.
- **Who may call.** *Admin* routes require a user whose policies include `AdministratorAccess`; anyone
  else gets `403`. *User* routes are open to every signed-in user, and the object-browser routes are
  further limited by the caller's IAM policies, as over S3. A session that still has to change its
  password gets `403` everywhere except `/admin/auth/*`.
- **Leader routing.** Writes to IAM, STS, SSO, buckets, storage classes, backup and restore, and the
  leader-only workers (lifecycle, bucket and site replication, rebalance) are forwarded to the leader
  automatically. Backup settings and history are always read from the leader.
- **Errors** are JSON: `{"error": "<HTTP reason>", "message": "<why>"}`. Validation failures answer
  `400`, missing things `404`, duplicates and state conflicts `409`, maintenance mode `503` with
  `Retry-After`.
- **Secrets are write-only.** Responses never contain a secret that was stored: backup reports
  `secretKeySet`, SSO reports `secretsSet`, and storage-class tier secrets, site-replication peer secrets,
  bucket-replication destination secrets, webhook headers and sensitive Kafka properties come back as
  `********`. Sending `********` back unchanged keeps the stored value.
- **Replication to the other API nodes.** A change is applied on the leader and then pushed to the
  other API nodes. If one of them does not acknowledge, the answer is still `200` and also carries
  `"replication": "pending"`, `unconfirmedNodes` (their node ids) and a `replicationMessage`; those
  nodes catch up on their own (see [IAM](../features/iam.md#at-rest-encryption-cluster-sync)).
- **Proxy parameters.** Some read routes accept `targetHost` / `targetPort` (and `nodeType=data` for
  logs) to be answered by a specific node. Only registered cluster nodes are accepted; anything else
  answers `400`.

## Authentication — `/admin/auth`

| Method | Path | Who | Purpose |
|---|---|---|---|
| POST | `/admin/auth/login` | anyone | Sign in; returns tokens |
| POST | `/admin/auth/refresh` | anyone | `{"refreshToken"}` → new token pair (refresh tokens are single-use) |
| POST | `/admin/auth/logout` | user | Revoke the current token |
| GET | `/admin/auth/session` | user | The session behind the token |
| POST | `/admin/auth/change-password` | user | `{"oldPassword","newPassword"}`; at least 8 characters, different from the old one |
| GET | `/admin/auth/sso/providers` | anyone | Enabled SSO providers for the login page |
| GET | `/admin/auth/sso/oidc/start`, `/oidc/callback`, `/saml/start`; POST `/saml/acs`, `/ldap/login` | anyone | Federated sign-in flows ([Single Sign-On](../features/sso.md)) |
| GET | `/admin/health` | anyone | `{"status":"UP","role":…,"leader":…}` once the node and the cluster are ready and a leader is known; otherwise `503` with `status` `STARTING` or `NO_LEADER` |

## Cluster

| Method | Path | Who | Purpose |
|---|---|---|---|
| GET | `/admin/stats` | user | API and data node counts |
| GET | `/admin/nodes/api`, `/admin/nodes/data` | user | Registered nodes with their payload (ports, disks, labels) |
| GET | `/admin/monitoring/metrics?startTs=&endTs=` | user | Per-node metrics history (served by the leader) |
| GET | `/admin/cluster/zookeeper[?targetHost&targetPort]` | user | ZooKeeper servers, quorum and the answering node's session |
| GET | `/admin/cluster/hrw[?targetHost&targetPort]` | user | One API node's HRW ownership counts |
| GET | `/admin/logs/tail?lines=&offset=[&targetHost&targetPort&nodeType]` | admin | Tail a node's log |

## Maintenance and workers — `/admin/maintenance`, `/admin/rebalance`

Reads are open to users; every `POST` is admin-only.

| Method | Path | Purpose |
|---|---|---|
| GET | `/admin/maintenance/status[?targetHost&targetPort]` | Background task status and `maintenanceMode` of a node |
| POST | `/admin/maintenance/mode` | `{"enabled": bool}` on every API node ([Maintenance Mode](../features/maintenance.md)) |
| POST | `/admin/maintenance/flush` | Flush write buffers cluster-wide |
| GET | `/admin/maintenance/repair/status` | Disk repair counters (`inProgress`, `totalRepaired`, `totalFailed`, `lastCycleStatus`, reactive queue) |
| POST | `/admin/maintenance/repair/scan` | Find dead disks and the shards they hold (`deadDisks`, `totalAffectedShards`) |
| POST | `/admin/maintenance/repair/trigger` | Run a repair cycle now |
| GET | `/admin/maintenance/scrubber/status` | Bitrot scrubber, summed over all API nodes, with a `nodes` array |
| POST | `/admin/maintenance/scrubber/enable` | `{"enabled": bool}` on every API node |
| POST | `/admin/maintenance/scrubber/trigger` | Scan now on every API node |
| GET | `/admin/maintenance/reconcile/status` | HRW reconcile sweep, summed over all API nodes, with a `nodes` array |
| POST | `/admin/maintenance/reconcile/enable`, `/reconcile/trigger` | On every API node |
| GET / POST | `/admin/maintenance/lifecycle/status`, `/enable`, `/trigger` | Lifecycle scanner (leader) |
| GET / POST | `/admin/maintenance/replication/status`, `/enable`, `/trigger` | Bucket replication worker (leader) |
| GET | `/admin/rebalance` | Chunk rebalance status (leader) |
| POST | `/admin/rebalance/enable`, `/admin/rebalance/run` | Toggle / run a rebalance cycle (leader) |

## Object browser — `/admin/browser`

Open to users, limited by their IAM policies. Data-changing calls answer `503` during maintenance.

| Method | Path | Required permission | Purpose |
|---|---|---|---|
| GET | `/admin/browser/buckets` | `s3:ListBucket` per bucket | Buckets the caller may list, with `objectCount`, `totalSizeBytes`, `versioning` |
| POST | `/admin/browser/buckets` | `s3:CreateBucket` | `{"name","versioning"}`; S3 naming rules, `409` if it exists |
| DELETE | `/admin/browser/buckets/{bucket}` | admin | `409` while it holds objects, `404` if missing |
| GET | `/admin/browser/objects/{bucket}` | `s3:ListBucket` | Current objects with their metadata |
| DELETE | `/admin/browser/objects/{bucket}/{key}[?versionId=]` | `s3:DeleteObject` | Delete (a delete marker on a versioned bucket) |
| POST | `/admin/browser/upload/{bucket}/{key}` | `s3:PutObject` | Body = object bytes, `Content-Type` = object type |
| GET | `/admin/browser/download/{bucket}/{key}` | `s3:GetObject` | Object bytes; `409` for SSE-C objects, `404` if missing |
| GET | `/admin/browser/versions/{bucket}/{key}` | `s3:ListBucketVersions` | Versions of a key |
| GET / PUT / DELETE | `/admin/browser/bucket-config/{bucket}/{type}` | admin | `type` = `policy`, `cors`, `lifecycle`, `replication`, `tagging`; documents are validated ([Bucket Configuration](../features/bucket-configuration.md)) |

Bucket object counts are exact and survive restarts: each API node counts the objects it is the HRW
primary for (reusing one metadata scan for `shannonstore.api.bucket.stats.cache.ms`) and the answering
node sums the nodes.

## Buckets and storage classes

| Method | Path | Who | Purpose |
|---|---|---|---|
| GET | `/admin/buckets/{bucket}/lifecycle` | user | Lifecycle XML (empty when none) |
| PUT / DELETE | `/admin/buckets/{bucket}/lifecycle` | admin | Set (validated) / remove |
| GET | `/admin/buckets/{bucket}/storage-class` | user | Assigned class and effective EC profile |
| PUT / DELETE | `/admin/buckets/{bucket}/storage-class` | admin | `{"storageClass": "..."}` / clear |
| GET | `/admin/storage-classes`, `/admin/storage-classes/{name}` | user | Registry (tier secret masked) |
| PUT / DELETE | `/admin/storage-classes/{name}` | admin | Create or update / delete (`409` while a bucket uses it) |

## IAM — `/admin/iam`

`GET` routes are open to users (a user sees only themselves and their groups); every other method is
admin-only.

| Method | Path | Purpose |
|---|---|---|
| GET | `/admin/iam/users` | Users with groups and access keys (secrets masked) |
| POST | `/admin/iam/users` | `{"userId","password","groups"}`; `409` if the user exists, password ≥ 8 characters |
| DELETE | `/admin/iam/users/{userId}`, `/admin/iam/users/{userId}/keys/{accessKey}` | Delete a user (not `admin`) / revoke a key |
| POST | `/admin/iam/keys` | `{"userId","expiresAt"?}` → `accessKey`, `secretKey`, `token` (shown once) |
| GET | `/admin/iam/download-key/{userId}/{accessKey}` | CSV with the key and secret (admins, or the key's own user) |
| GET / POST | `/admin/iam/groups` | List / create (`409` if it exists) |
| DELETE | `/admin/iam/groups/{group}` | Delete |
| GET / POST | `/admin/iam/policies` | List / create or replace `{"name","json"}` (JSON with a `Statement`) |
| DELETE | `/admin/iam/policies/{name}` | Delete (not `AdministratorAccess`) |
| POST | `/admin/iam/add-user-to-group`, `/remove-user-from-group` | `{"userId","groupName"}` |
| POST | `/admin/iam/attach-group-policy`, `/detach-group-policy` | `{"groupName","policyName"}` |
| POST | `/admin/iam/attach-user-policy` | `{"userId","policyName"}` |

Unknown users, groups and policies in these calls answer `404`.

## STS — `/admin/sts` (admin)

| Method | Path | Purpose |
|---|---|---|
| GET | `/admin/sts/sessions` | Temporary credentials (no secrets) |
| POST | `/admin/sts/sessions` | `{"parentAccessKey","durationSeconds","sessionName"}` → full credentials, once |
| GET | `/admin/sts/sessions/download/{accessKeyId}` | CSV of one session |
| DELETE | `/admin/sts/sessions` | `{"accessKeyId"}`; the revocation reaches every API node |
| POST | `/admin/sts/sessions/clear` | Remove expired sessions |

## KMS — `/admin/kms` (admin)

| Method | Path | Purpose |
|---|---|---|
| GET | `/admin/kms/list` | Key IDs with current version and last rotation |
| GET | `/admin/kms/status/{keyId}` | Versions (number, state, created) — never key material |
| POST | `/admin/kms/create` | `{"keyId"}`; `409` if it exists |
| POST | `/admin/kms/rotate/{keyId}` | New ACTIVE version; `404` for an unknown key |

## SSO settings — `/admin/sso` (admin)

| Method | Path | Purpose |
|---|---|---|
| GET / PUT | `/admin/sso/settings` | Stored settings (secrets as `secretsSet`) / update |
| GET | `/admin/sso/saml/metadata` | SP metadata XML |
| POST | `/admin/sso/saml/import-metadata` | `{"metadata": "<IdP XML>"}` |
| POST | `/admin/sso/saml/generate-keypair` | New SP keypair (returns the certificate) |
| POST | `/admin/sso/test/{oidc|saml|ldap}` | Connection test with the saved settings |

## Notifications and site replication (admin, except stats)

| Method | Path | Purpose |
|---|---|---|
| GET / PUT | `/admin/notifications/config` | Targets, rules, master switch (secrets masked) |
| POST | `/admin/notifications/test` | `{"targetId"}` → `{"ok": true}` or `{"ok": false, "error"}` |
| GET | `/admin/notifications/stats` | Delivery counters of the answering node (user) |
| GET / PUT | `/admin/replication/sites` | `{"config": "<json>"}` / `{"enabled","peers":[...]}` (each peer needs a unique `siteId` and an `http(s)://` endpoint) |
| GET | `/admin/replication/sites/status` | Leader's site-replication counters (user) |
| POST | `/admin/replication/sites/trigger` | Run a cycle now |

## Backup and restore (admin)

| Method | Path | Purpose |
|---|---|---|
| GET / POST | `/admin/backup/config` | Settings (`secretKeySet` instead of the secret) / update — an invalid cron, or enabling without endpoint, bucket and credentials, answers `{"status":"error","message"}` |
| POST | `/admin/backup/test-connection` | Probe the target with optional unsaved overrides |
| POST | `/admin/backup/run` | Start a backup in the background → `{"status":"started"}`; `409` while one runs |
| GET | `/admin/backup/history` | `inProgress` and recent runs |
| POST | `/admin/restore` | `{"backupId"}` → staging starts in the background |
| GET | `/admin/restore/status` | `inProgress` (backup id) and the `last` restore report |

See [Backup & Restore](../features/backup.md).
