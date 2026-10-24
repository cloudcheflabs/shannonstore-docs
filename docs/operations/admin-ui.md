# Admin UI Guide

The Admin UI is the web console every API server serves on its admin port (default `8888`) under
`/admin/`. Behind the usual nginx in front of the API servers it is `http://<nginx>/admin/`. Any API
server answers it; writes that must run on the leader are forwarded to the leader automatically, so it
does not matter which node the browser reaches.

This page walks through every page and control. The REST calls behind each control are listed in the
[Admin REST API](admin-rest-api.md) reference.

## Signing in

| Control | What it does |
|---|---|
| **User ID / Password** | Local account sign-in (`POST /admin/auth/login`). A wrong password shows *Invalid credentials*. |
| **Use your directory account instead** | Shown when LDAP is enabled: the same form signs in against the directory. |
| **Single Sign-On** / **Sign In with SAML** | Shown when OIDC / SAML is enabled: redirects to the identity provider. |

**First login.** A new cluster has one account, `admin` / `admin`, flagged to change its password. Until it
is changed the console shows only the *Password Change Required* dialog and the server refuses every other
admin API with `403`. The new password must be at least 8 characters and different from the current one.

**Header.** The top bar shows the signed-in user and an *Administrator* or *User* badge, a **lock** button
that opens *Update Password* (any user can change their own password there) and the **sign-out** button.

**Session.** Sessions are short-lived JWTs that the console refreshes in the background. A forged or
expired token sends you back to the login form. Reloading or bookmarking any console page works: the
server answers browser navigations with the console itself.

## Who sees what

Administrators (users whose policies include `AdministratorAccess`) see every page. Other users see
**Dashboard**, **Topology**, **HRW Placement**, **Object Store** and **Identity Control**; the remaining
pages are hidden from the sidebar and show *Administrator access required* when opened directly. The
server enforces the same split — the admin-only routes answer `403` to a non-administrator whatever
client calls them.

What a user can do in the **Object Store** is decided by their IAM policies, exactly as over the S3 API:
they see only buckets their policies let them list, and upload, download and delete only where
`s3:PutObject`, `s3:GetObject` and `s3:DeleteObject` allow.

## Dashboard

Real-time health of the cluster, refreshed every 10 seconds.

- **API Servers / Data Nodes** cards — the number of registered nodes. Clicking a card switches the charts
  to that node type.
- **Overall / API Servers / Data Nodes** toggle — *Overall* shows the API and data averages; the other
  two draw one line per node.
- Charts: **CPU Usage (%)**, **Memory Usage (MB)**, **S3 Throughput (Req)** and **S3 Latency (ms)**
  (the S3 charts are hidden in the data-node view), over the last 30 minutes.
- **Disk Usage – Data Nodes** — every data disk with its used/free space; a bar turns amber above 70 %
  and red above 85 %.

## Topology

The cluster's nodes, fault domains and ZooKeeper ensemble, refreshed every 5 seconds.

- **Topology** — counts of distinct zones, racks and hosts among the data nodes and data nodes
  themselves. *explicit* means nodes carry the label (`hostId` / `rackId` / `zoneId`); *fallback* means
  it is derived (rack from host, host from the registration address).
- **API Servers** — node id, address, topology labels, S3 and admin ports, and a *MAINTENANCE* tag
  while maintenance mode is on.
- **Data Nodes** — node id, address, labels, total capacity and per-disk usage.
- **Logs** (administrators) — opens the **Log Stream** of that node: the last 100 lines, then new lines
  every 2 seconds. **LIVE / PAUSED** stops and resumes streaming, the refresh button reloads from the
  start, **X** closes. Data-node logs are fetched over the internal NIO channel.
- **ZooKeeper** — every server in `shannonstore.zk.connect` with its role (leader / follower /
  standalone / observer), version, connections, latency, zxid and status, the **Quorum OK / No Quorum**
  badge, and each API node's own ZooKeeper session (CONNECTED / SUSPENDED / LOST).

**Global Actions** (administrators):

| Button | Effect |
|---|---|
| **Disk Repair** | Scans for dead disks; if any, triggers a repair cycle and follows its progress until it finishes. On a healthy cluster it reports *No Repairs Needed*. |
| **Enter Maintenance** | After confirmation, every API node refuses S3 requests and data-changing admin actions with `503` (see [Maintenance Mode](../features/maintenance.md)). A red banner and the *MAINTENANCE* tags show it; the setting survives restarts. |
| **Exit Maintenance** | Shown instead while maintenance is on (also on the banner): resumes traffic on every node. |

## HRW Placement

How metadata ownership is spread over the API nodes by rendezvous hashing.

- Cards: **Replication Mode**, **Replication Factor (R)**, **Cluster-wide Local Entries** and
  **Foreign-Owned Local Copies** (entries a node holds but no longer owns).
- **Per-Node Ownership** — for each API node: local entries, owned, primary, foreign, and the owned
  ratio against the expected share (`R / nodes`).
- **Reconciliation Sweep** — the background job that removes foreign copies once every current owner
  holds them. The panel sums the counters over all API nodes and lists each node's own switch and state.
  **Enable Sweep / Disable Sweep** and **Trigger Cycle Now** (administrators) apply to **every** API node.
  *DRY-RUN* means the cluster only logs what it would delete.
- **Refresh** reloads everything.

## Object Store

A browser over buckets and objects.

**Buckets (left column)**

- Each bucket shows its object count and size, and a *Versioning* badge when versioning is on. Counts are
  exact: each API node counts the objects it is the HRW primary for and the console sums them.
- **New Bucket** opens *Provision New Bucket*: a name that follows the S3 rules (3–63 characters,
  lower-case letters, digits, dots and hyphens) and the **Object Versioning** switch. An invalid or
  duplicate name is reported inside the panel.
- On hover (administrators): the **gear** opens the bucket configuration panel and the **trash** button
  deletes the bucket after confirmation. A bucket that still holds objects is refused with *not empty*.

**Objects (right side)**

- Objects are shown as folders and files by `/` in the key. Click a folder to open it; the breadcrumb and
  the `..` row go back up.
- **Upload** sends one or more files into the current folder (`POST /admin/browser/upload/...`), with the
  same encryption, compression and erasure coding as an S3 `PutObject`. The size is limited by
  `shannonstore.api.network.http.max.content.length` (256 MiB by default); use an S3 client for large objects.
- **Refresh** reloads the listing.
- **Download** saves the object under its own name. Objects written with a customer key (SSE-C) cannot
  be opened from the console and say so.
- The **trash** button on a row deletes the object after confirmation (on a versioned bucket this
  writes a delete marker).

**Bucket configuration panel** (administrators) — one tab per S3 sub-resource, each with **Save** and
**Delete**:

| Tab | Content |
|---|---|
| **Policy (JSON)** | Bucket policy. Must be a JSON object with a `Statement`; anything else is refused with a message and the stored policy is kept. |
| **CORS (XML)** | `<CORSConfiguration>` with at least one `<CORSRule>` carrying `<AllowedOrigin>` and `<AllowedMethod>`. |
| **Lifecycle (XML)** | `<LifecycleConfiguration>` with at least one well-formed `<Rule>` whose `<Status>` is `Enabled` or `Disabled`. Malformed XML is refused and the existing rules are kept. |
| **Replication (XML)** | `<ReplicationConfiguration>` whose rules name a `<Destination><Bucket>`. A stored destination `<SecretKey>` is shown as `********`; leave it to keep the stored secret. |
| **Tags** | Key/value rows: **Add tag**, the trash button per row, then **Save**. |

## Identity Control

Users, groups, policies and STS sessions. Non-administrators see only their own user.

**Users**

- **New User** — user ID (letters, digits and `. _ @ + -`, up to 64) and a password of at least 8
  characters. Creating a user that already exists is refused (it can no longer replace and demote an
  existing account, including `admin`).
- **Join** adds the user to a group.
- **New Keypair** issues an access key, optionally with an expiration date. The secret is shown once in a
  banner; afterwards it is available only through the **download** button on the key, which saves a CSV
  with the key and secret. The **X** on a key revokes it after confirmation.
- The **trash** button deletes the user (not offered for `admin`).

**Groups**

- **New Group** creates a group; duplicates are refused.
- Click a row to expand it: its members (hover a member for **X** to remove it) and its policies (the
  **X** on a policy detaches it). **Attach Custom** attaches a policy.
- The **trash** button deletes the group.

**Policies**

- **New Policy** — a name, an optional **Governance Template** (*AdministratorAccess*,
  *ReadWriteAccess*, *ReadOnlyAccess*) that fills the JSON, and the IAM JSON itself. A document that is
  not valid JSON or has no `Statement` is refused with the reason.
- On hover, the **pencil** edits a policy (the name is locked) and the **trash** deletes it.
  *AdministratorAccess* can be edited but not deleted.

**STS Sessions** (administrators)

- **Generate Temporary Credentials** — a parent access key, a lifetime in hours and minutes and an
  optional session name. The access key, secret and session token are shown once (the eye buttons reveal
  them, the copy buttons copy them) with **Download CSV**.
- The table lists sessions with their remaining lifetime and *Active / Expired* state; the download
  button saves a session's CSV and the trash button revokes it. A revoked session stops working on every
  API node. **Clear Expired** removes expired sessions.

## Security & KMS

(Administrators.) The cluster key-encryption keys.

- **Create New Master Key** — a key ID (letters, digits and `. _ -`) and **Generate Key**, confirmed in a
  dialog. Creating an ID that already exists is refused; use rotation instead.
- The key table shows each KEK's current version and last rotation. **Manage Key** opens its
  **Version Lifecycle**: every version with its state (*ACTIVE* encrypts and decrypts, *RETIRED* only
  decrypts) and creation time.
- **Rotate Key Now** adds a new ACTIVE version and retires the previous one. The rotation reaches every
  API node and data node.
- **Back to List** and the refresh button return to and reload the list.
- **Change Admin Password** opens the password dialog.

Key material never leaves the cluster: the console and the API show version metadata only.

## Single Sign-On

(Administrators.) OIDC, SAML 2.0 and LDAP/Active Directory for both the console and the S3 API; see
[Single Sign-On](../features/sso.md) for the concepts.

- Tabs **OpenID Connect**, **SAML 2.0** and **LDAP / Active Directory**; a green dot marks a provider
  that is running.
- Each tab has its enable switch and fields. Secrets (OIDC client secret, LDAP bind password, SAML SP
  private key) are write-only: the field says *stored* and a blank field on save keeps the stored value.
- SAML extras: **Download SP metadata** (the XML to give your IdP), **Generate keypair** (an SP
  certificate and private key) and **Read metadata** (paste the IdP metadata XML to fill entity ID,
  sign-on URL and certificate).
- **Identity mapping** — group mappings, federated session length and *Allow sign-in with no mapped
  group*.
- **Save** applies the settings on every node, **Test connection** checks the selected provider with the
  saved settings, **Discard changes** reloads the stored values.

## Maintenance pages

All administrator-only. Each page polls its status every few seconds and has a **Refresh** button.

### Bitrot Scrubber

Verifies the CRC32C of every shard and rebuilds corrupt ones from parity
([Data Integrity](../features/data-integrity.md)). The scrubber runs on every API node over the metadata
that node holds, so **Enable / Disable Scrubber** and **Trigger Scan Now** apply to all API nodes, the
counters (scanned, corrupt, repaired, failed) are summed, and **Per API node** lists each node's switch,
progress and last cycle.

### Lifecycle Expiry

Runs bucket lifecycle rules: expiration by days, seconds or date, noncurrent-version expiration and
abort of stale multipart uploads, skipping WORM-protected objects. It runs on the leader; the page always
shows the leader's view (*Leader Node: Yes*). **Enable / Disable Scanner** and **Trigger Scan Now**.

### Storage Classes

The cluster registry of storage classes ([Storage Classes & Tiering](../features/storage-classes-tiering.md)).

- **New Class** / **Edit** — name, description, EC data and parity shards (the editor shows the storage
  overhead) and an optional tier destination (endpoint, region, bucket, KMS key, access key, secret key)
  used by lifecycle `Transition` rules. A stored tier secret is shown as `********`; saving it unchanged
  keeps the stored secret.
- The **trash** button deletes a class after confirmation; a class that buckets still use is refused.

### Chunk Rebalance

Moves chunks to the disks the placement cascade picks today, after a topology change. It runs on the
leader, and the page and its switches always talk to the leader. **Enable / Disable** and **Run now**;
the status shows cycles, objects scanned and chunks migrated. On a cluster whose topology has not changed
a cycle migrates nothing.

### Bucket Replication

Copies objects of buckets with a replication configuration to their destination
([Bucket Configuration](../features/bucket-configuration.md)). Leader-only. **Enable / Disable
Replication** and **Replicate Now**; counters for replicated objects, delete markers and failures.

### Site Replication

Active-active replication of every bucket to peer clusters ([Site Replication](../features/site-replication.md)).

- Peer rows: site ID, endpoint (`http://` or `https://`), access key, secret key and region; the trash
  button removes a row, **Add peer** adds one.
- **Save Peers** stores the peers; **Enable / Disable** stores them together with the switch;
  **Replicate Now** runs a cycle on the leader.
- A stored peer secret is shown as `********`; leave it to keep the stored secret.

### Event Notifications

Object events to webhooks or Kafka ([Object Event Notifications](../features/event-notifications.md)).

- **Turn On / Turn Off Notifications** — the master switch (applied by **Save Configuration**).
- **Add Webhook / Add Kafka** — a target with its ID, enabled box, URL (and optional `Authorization`
  header) or bootstrap servers and topic. **Test** sends a test event to the target; the trash button
  removes it.
- **Add Rule** — routes events to a target by bucket (`*` = all), prefix, suffix and event types.
- **Save Configuration** stores everything cluster-wide.
- **Delivery Stats (this node)** — queue depth, emitted, delivered, retried, dropped and failed events.
- Stored header values and sensitive Kafka properties are shown as `********`; saving them unchanged keeps
  the stored values.

### S3 Backup

Backs up the whole cluster state to an external S3 target and restores it ([Backup & Restore](../features/backup.md)).

- **Configuration** — enable switch, S3 endpoint, region, bucket, prefix, access key, secret key
  (write-only; *unchanged* means one is stored), interval, retention, optional cron, path-style
  addressing. **Save** refuses an invalid cron expression and refuses to enable backups without an
  `http(s)://` endpoint, a bucket and credentials.
- **Test connection** checks the target with the values on the form, saved or not.
- **Backup now** starts a backup on the leader and returns at once; the run appears under *In progress*
  and then in **History**.
- **History** — each run's time, trigger (manual, scheduled, cron), status, files, uploaded and reused
  bytes. **Restore** on an OK run stages that backup on the leader and every data node in the background;
  when staging finishes the report shows per-node results. The restored state takes effect on the next
  restart of the API servers and data nodes.

Backup settings and history live on the leader; every node shows the leader's.
