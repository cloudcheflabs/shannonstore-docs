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
| **Socket path** | `data/s3-metadata/admin.sock` (mode `600`) |
| **Authentication** | OS file permission — same user as the api node process |
| **Network surface** | none — Unix domain socket only |
| **Downtime** | none — applied in-process on the live api node |
| **Cluster sync** | automatic — IAM state propagates to peer nodes via the existing sync path |
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

## Configuration

The recovery socket is enabled by default. Two layers resolve the socket path: the
`bin/shannonstore-cli.sh` wrapper script sets `SHANNONSTORE_ADMIN_SOCKET` if it isn't
already set, then the Java CLI itself reads `--socket` / that env var:

1. `--socket /path/to/admin.sock` command-line flag — checked by the Java CLI, highest priority, always wins.
2. `SHANNONSTORE_ADMIN_SOCKET` environment variable, if already set in the caller's shell before invoking the script.
3. Otherwise, the wrapper script prefers the path the *running* api node published to `bin/api.socket` (written by the server itself at startup) — this is the one authoritative source when the IAM RocksDB dir was overridden with a `-D` flag or edited after startup, since it reflects whatever the live process actually bound.
4. If that marker file is missing or stale, the wrapper falls back to the first existing socket among:
   - `<base_dir>/data/s3-metadata/admin.sock`
   - `<base_dir>/data/admin.sock`
   - `/data/s3-metadata/admin.sock`
   - `/data/admin.sock`
5. Final fallback: `<base_dir>/data/s3-metadata/admin.sock`

The api node creates the socket beside its IAM directory: with the default
`shannonstore.api.iam.rocksdb.dir = ./data/s3-metadata/iam`, the socket
lives at `./data/s3-metadata/admin.sock` and the audit log at
`./data/s3-metadata/iam-audit/reset.log`. If you override the IAM RocksDB
dir, both paths move with it.

## Security model

**1. The socket is OS-gated.**
At startup the api node creates `data/s3-metadata/admin.sock` with mode
`600` (owner read/write only). Even other unprivileged users on the same
host cannot connect. There is no token, no shared secret, no network
listener.

**2. The audit log records every reset.**
Every successful reset appends a JSON line to
`data/s3-metadata/iam-audit/reset.log` (mode `600`). The plaintext
password is **never** logged — only the first 8 characters of its hash,
the user, whether it was server-generated, and the OS user that invoked
the CLI.

```json
{"ts":"2026-05-23T16:00:46.922Z","event":"iam.reset-password","user":"admin","generated":true,"hashFp":"OYV/Ojf/","invokedAs":"root"}
```

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
