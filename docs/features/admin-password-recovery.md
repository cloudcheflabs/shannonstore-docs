# Admin Password Recovery

ShannonStore ships with a built-in recovery channel that lets an operator
reset the `admin` user's password **without stopping the api node**, even
when no one remembers the current password.

Recovery uses a local Unix domain socket — there is no HTTP back-door, no
network endpoint, no recovery URL. Authentication is performed by the
operating system: only a process that already shares the api node's
filesystem identity can open the socket.

## When to use

- The admin password was forgotten or rotated out of the password manager.
- An automation script needs to provision a known admin password during
  first-time setup, without going through the web UI.
- A new operator is onboarded and you need to hand them an admin credential.

For day-to-day password changes (the user remembers the old password and
wants a new one), use the Admin UI's **Change Password** screen instead —
that path requires the old password and does not flag the user for forced
rotation.

## How it works

```
┌────────────────────┐   JSON over UDS    ┌────────────────────┐
│  shannonstore-cli  │ ─────────────────▶ │  Api node (running)│
│  iam:reset-password│                    │   AuthManager      │
└────────────────────┘                    │   .adminResetPwd() │
         ▲                                │                    │
         │ stdout: new password           │  saveToLocalDb()   │
         │ (one-time)                     │  cluster sync push │
                                          │  audit log append  │
                                          └────────────────────┘
```

| Property | Value |
|---|---|
| **Socket path** | `./data/s3-metadata/admin.sock` by default, mode `600` — the live value is published to `bin/api.socket` |
| **Authentication** | OS file permission — same user as the api node process |
| **Socket path marker** | `bin/api.socket` — written when the socket binds, removed on shutdown |
| **Network surface** | none — Unix domain socket only |
| **Downtime** | none — applied in-process on the live api node |
| **Cluster sync** | leader-owned — run on a follower, the reset is forwarded to the leader, applied there, pushed to every api node, and confirmed on the follower before the CLI exits 0 |
| **Audit log** | `data/s3-metadata/iam-audit/reset.log` (mode `600`, append-only) |
| **Post-reset state** | `requirePasswordChange = true` (forced rotation on next login) |

## Quick start

The simplest invocation lets the api node generate a strong 20-character
password and print it to stdout. The new password must be changed on the
admin's next login (the `requirePasswordChange` flag is set automatically).

```bash
# Inside the api node host or container:
bin/shannonstore-cli.sh iam:reset-password
```

### After the reset: change the password

The reset sets `requirePasswordChange = true`. Until the password is changed, every admin
API except login and change-password answers `403` with
*Password change required before accessing this API: this account's password is the initial
one or was reset by an administrator. Change it with POST /admin/auth/change-password …*

The CLI says so on **stderr** — stdout stays exactly the generated password, so
`NEW=$(bin/shannonstore-cli.sh iam:reset-password)` keeps working — and prints the exact calls
for this node:

```text
note: 'admin' must change this password before any other admin API call works (until then they answer 403 "Password change required"). Log in with it, then change it:
  curl -s -X POST http://localhost:8888/admin/auth/login -H 'Content-Type: application/json' -d '{"userId":"admin","password":"<password above>"}'      # -> accessToken
  curl -s -X POST http://localhost:8888/admin/auth/change-password -H 'Content-Type: application/json' -H 'Authorization: Bearer <accessToken>' -d '{"oldPassword":"<password above>","newPassword":"<new password>"}'
  (or sign in to the Admin UI, which asks for the new password.)
```

## Input modes

| Mode | Command | When to use |
|---|---|---|
| **Server-generated** | `shannonstore-cli.sh iam:reset-password` | Default. Strong random password printed once on stdout. |
| **Explicit** | `shannonstore-cli.sh iam:reset-password --new-password 'My!Pass'` | Automation that knows the desired value. Beware: argv may show up in `ps`. |
| **Stdin** | `echo 'My!Pass' \| shannonstore-cli.sh iam:reset-password --new-password -` | Automation that wants to avoid argv exposure. |
| **Interactive** | `shannonstore-cli.sh iam:reset-password --interactive` | Operator at a TTY. Prompts for password twice with no echo. |

Resetting a different user is also supported:

```bash
bin/shannonstore-cli.sh iam:reset-password --user some-user --new-password 'NewPass123'
```

## Running it in a cluster (leader / follower)

IAM is owned by the **leader** api node: the leader applies every change and pushes the
IAM state to the other api nodes. You can run the CLI on **any** api node:

| Where you run it | What happens | Exit code |
|---|---|---|
| Leader | Applied on the leader, pushed to every api node | `0` |
| Follower, leader healthy | Forwarded to the leader over the internal NIO port, applied there, then confirmed on this follower (`replicatedHere`) | `0` |
| Follower, no leader elected | Nothing is applied anywhere | `1` |
| Follower, leader paused / unreachable / too slow | Nothing is applied anywhere; the message names the leader | `1` |

A failure always says **the password was NOT changed** and names the leader, for example:

```text
error: this api node (8cf0…) is a follower and the leader a272… (api-1:9000) did not apply
the reset (no answer within 15000 ms); the password was NOT changed. (A leader discards a
forwarded reset that reaches it after 2026-10-27T11:47:14.331Z.) Check the leader, or run
shannonstore-cli on the leader host.
```

The forwarded request carries a **deadline** (one third of the forward timeout). A leader
that receives it later — it was frozen by a GC pause, `docker pause`, or a network
partition — discards it, so a reset the CLI reported as failed can never be applied
afterwards. Before this, a reset on a follower changed only that follower's copy, the
leader's next IAM push silently put the old password back, and the CLI still exited `0`.

`ping` shows the node's role and the current leader, so you can tell where you are:

```bash
bin/shannonstore-cli.sh ping
# pong role=follower node=8cf0d2eb-… leader=a272d87d-… (api-1:9000)
```

The request and the answer travel KMS-encrypted on the internal port (they can carry a
password), on their own message type.

## Configuration

The recovery socket is enabled by default. Every key below lives in
`conf/shannonstore.properties` and is read at api node startup:

```properties
# conf/shannonstore.properties
# Set false to remove the local recovery path entirely.
shannonstore.api.admin.socket.enabled      = true
# Leave EMPTY to derive the path from the IAM store
# (<parent of shannonstore.api.iam.rocksdb.dir>/admin.sock), which is the default.
shannonstore.api.admin.socket.path    =
# Name of the file under <shannonstore.home>/bin that receives the socket path the
# api node actually bound to (see "How the CLI finds the socket" below).
shannonstore.api.admin.socket.marker.file  = api.socket
# Follower only: how long to wait for the leader to apply a forwarded reset before the
# CLI fails with "password was NOT changed". The leader discards a forwarded reset that
# arrives later than one third of this.
shannonstore.api.admin.socket.forward.timeout.ms = 15000
# Append-only audit trail of socket operations.
shannonstore.api.iam.audit.dir = <socket dir>/iam-audit
```

With the shipped `shannonstore.api.iam.rocksdb.dir = ./data/s3-metadata/iam`, the socket
lands at `./data/s3-metadata/admin.sock` and the audit log at
`./data/s3-metadata/iam-audit/reset.log`. Setting `shannonstore.api.admin.socket.path`
moves the socket without moving the IAM store; leaving it empty keeps the two together.

### How the CLI finds the socket

Re-deriving the socket path from `conf/shannonstore.properties` is not reliable on its own:
`shannonstore.base.data.dir` can be overridden with `-D` at launch or edited after
startup, and the file does not record which value the live process used. So the
api node **publishes the path it actually bound to** into
`<install dir>/bin/api.socket` when the socket comes up, and removes that file on
shutdown. `bin/shannonstore-cli.sh` prefers it.

Full resolution order, highest priority first:

1. `--socket /path/to/admin.sock` — read by the Java CLI, always wins.
2. `$SHANNONSTORE_ADMIN_SOCKET` — if already exported in the caller's shell.
3. `<install dir>/bin/api.socket` — the path published by the running api node.
   Used only when the file exists *and* the path in it is a live socket.
4. `shannonstore.api.admin.socket.path` from `conf/shannonstore.properties`, with
   `${shannonstore.base.data.dir}` expanded. A value that still contains a
   `${...}` placeholder is rejected rather than used literally.
5. `<install dir>/data/admin.sock`, then `/data/admin.sock`.

Step 3 is what makes a moved data dir work: with the socket at
`/data/admin.sock` and the properties file still saying `./data`, only the marker
knows where to connect.

To rename the marker, change one key — both ends read it:

```bash
# conf/shannonstore.properties
shannonstore.api.admin.socket.marker.file = shannonstore-recovery.socket
```

Restart the api node; it publishes `bin/shannonstore-recovery.socket`, and the CLI
picks the new name up from the same properties file.

### The master key is for the api node, not the CLI

`SHANNONSTORE_MASTER_KEY` must be exported for the api node process. `bin/start-api-server.sh` checks it up front and refuses to start when it is unset.
The variable name itself is configurable — `shannonstore.kms.master.key.env` in
`conf/shannonstore.properties` names the variable the api node reads:

```bash
export SHANNONSTORE_MASTER_KEY='replace-with-a-32-char-or-longer-secret'
bin/start-api-server.sh
```

`bin/shannonstore-cli.sh` does **not** need it. The CLI only opens the Unix socket and
hands the request to the running api node, which already holds the unsealed key,
so this works with the variable unset:

```bash
unset SHANNONSTORE_MASTER_KEY
bin/shannonstore-cli.sh ping
# pong
```

If a CLI invocation complains about the key rather than the socket, you are
running a start script, not the CLI.

### Worked examples

```bash
# 1. On the host, as the same OS user that runs the api node:
cd /opt/shannonstore
bin/shannonstore-cli.sh ping
bin/shannonstore-cli.sh iam:reset-password

# 2. The api node runs as a service account and you are root:
sudo -u shannonstore /opt/shannonstore/bin/shannonstore-cli.sh iam:reset-password

# 3. Inside a container:
docker exec -it shannonstore-api-1 /app/bin/shannonstore-cli.sh iam:reset-password

# 4. Data dir was relocated at launch — no extra flags needed, the CLI
#    reads the published marker:
cat /opt/shannonstore/bin/api.socket
# /var/lib/shannonstore/admin.sock
bin/shannonstore-cli.sh ping

# 5. Socket in a non-standard place and no marker (the api node is stopped,
#    or you are on a host where the marker was cleaned up):
bin/shannonstore-cli.sh --socket /var/lib/shannonstore/admin.sock iam:reset-password

# 6. Non-interactive automation, password from stdin so it never reaches argv:
echo 'S0me!Strong!Pass' | bin/shannonstore-cli.sh iam:reset-password --new-password -
```

## Security model

**1. The socket is OS-gated.**
At startup the api node creates `data/s3-metadata/admin.sock` with mode
`600` (owner read/write only). Even other unprivileged users on the same
host cannot connect. There is no token, no shared secret, no network
listener.

**2. The audit log records every reset.**
Every successful reset appends a JSON line to
`data/s3-metadata/iam-audit/reset.log` (mode `600`). Nothing derived from the
plaintext password is logged. `hashFp` is the first 12 hex characters of SHA-256
over the **stored, salted** password hash — it identifies the password record (the
leader and a follower print the same value for the same reset), but it cannot be
used to test password guesses, and resetting to the same password twice gives two
different values. `route` shows where a forwarded reset came from or went to.

```json
{"ts":"2026-10-27T11:46:35.578Z","event":"iam.reset-password","user":"admin","generated":false,"hashFp":"948b633b6622","invokedAs":"root","route":"forwardedFrom=8cf0d2eb-…"}
```

Earlier versions logged a prefix of the **unsalted** SHA-256 of the plaintext password,
which let anyone with read access to the log check password guesses offline. Lines written
by those versions are still in old logs; rotate the affected passwords if the log may have
been read.

**3. The new password is exposed exactly once.**
For server-generated passwords, the plaintext is returned only on the
single CLI invocation that triggered the reset. It is not retransmitted.
Treat scrollback and shell history accordingly — or use stdin input mode
to avoid argv exposure entirely.

**4. Forced rotation on next login.**
After reset, the user is flagged `requirePasswordChange = true`. The next
successful login forces the user through the change-password flow, so a
temporary password used by the operator is immediately replaced by a
password only the user knows.

**5. The api node must be running.**
Because the recovery channel is in-process, the api node must be alive
for the CLI to connect. This is intentional: RocksDB requires an exclusive
lock, so an offline edit would either conflict with a running node or
need a complex stale-lock recovery. With this design, the only way to
reset is to be on the host *and* have the api node running *and* share
its filesystem identity.

## Limitations

- **Api node must be running.** If the api node is down, this CLI cannot
  help. Bring the api node back up first, then run the CLI.
- **No knowledge factor.** Any process that shares the api node's filesystem
  identity can invoke the CLI. In multi-tenant or shared-shell
  environments, restrict shell access to the api node accordingly. A
  future enhancement may add an opt-in "recovery key" requirement for an
  additional knowledge factor.

## Related

- [Identity & Access Management](iam.md)
- [Authentication & Authorization](auth-authz.md)
- [Distributed Architecture](distributed-architecture.md)
