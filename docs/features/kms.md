# Encryption & Key Management

ShannonStore encrypts every byte of object data and every byte of cluster metadata at rest. The implementation uses **envelope encryption** with AES-256-GCM throughout, but it is a **three-tier** hierarchy, not a simple "master key wraps the DEK directly" model: a master *password* protects a local keystore, the keystore holds one or more randomly generated, independently versioned Key Encryption Keys (KEKs), and it's those KEKs — not the master password itself — that wrap each object's Data Encryption Key (DEK).

```text
   ┌──────────────────────────────────────────┐
   │  SHANNONSTORE_MASTER_KEY  ← env var, never on disk
   │   (a passphrase, not used as an AES key directly)
   └──────────────┬───────────────────────────┘
                  │  PBKDF2WithHmacSHA256 (200,000 iters, random 32-byte salt)
                  ▼
   ┌──────────────────────────────────────────┐
   │  Derived wrapping key (AES-256-GCM)      │
   └──────────────┬───────────────────────────┘
                  │  encrypts/decrypts
                  ▼
   ┌──────────────────────────────────────────┐
   │  Keystore blob in KMS RocksDB:           │
   │  keyId → KeyLifecycle { versions:        │
   │    version# → KeyVersion(key, state) }   │
   │  each KeyVersion is a random AES-256 KEK,│
   │  state ∈ ACTIVE / RETIRED / REVOKED       │
   └──────────────┬───────────────────────────┘
                  │  ACTIVE KEK wraps
                  ▼
   ┌──────────────────────────────────────────┐
   │  Data Encryption Keys (DEK) — per object │
   │   AES-256-GCM, 96-bit IV per shard       │
   │   wrapped DEK + KEK version# persisted   │
   │   with object metadata                   │
   └──────────────┬───────────────────────────┘
                  │  encrypts
                  ▼
   ┌──────────────────────────────────────────┐
   │  Object shards on disk (EC + compressed) │
   └──────────────────────────────────────────┘

   ┌──────────────────────────────────────────┐
   │  Metadata stores (IAM, BucketManager,    │
   │  KMS RocksDB) — encrypted with the       │
   │  default-keyId KEK on every write         │
   └──────────────────────────────────────────┘
```

## Layers

### Master key (`SHANNONSTORE_MASTER_KEY`)

A passphrase passed in through the environment:

```bash
export SHANNONSTORE_MASTER_KEY="…32 ASCII chars or longer…"
```

It never leaves process memory after start, is **not** written to RocksDB, not logged, not surfaced through any admin endpoint. It is *not* used as an AES key directly — `ClusterKmsProvider` runs it through `PBKDF2WithHmacSHA256` (200,000 iterations by default, a random 32-byte salt persisted alongside the keystore blob) to derive the AES-256-GCM key that wraps/unwraps the keystore blob in KMS RocksDB. Losing the master key permanently locks the keystore, and therefore every KEK inside it, and therefore every encrypted blob the cluster has on disk.

The master key is required at startup. The API node refuses to launch without it (`ClusterKmsProvider` throws a `SecurityException` from its constructor if the env var is unset or blank), so a misconfigured deployment fails closed rather than running with no encryption.

### Key Encryption Keys (KEK)

The keystore itself holds one or more **randomly generated** 256-bit AES KEKs, keyed by `keyId` (e.g. `default`, `default-sse-s3-key` — auto-created the first time a node becomes leader with an empty keystore). Each `keyId` is a `KeyLifecycle` with its own map of `version → KeyVersion`, and each `KeyVersion` carries a state: `ACTIVE`, `RETIRED`, or `REVOKED`. The master password never wraps a DEK directly — it only protects the keystore blob that these KEKs live in.

### Data Encryption Keys (DEK)

For each object (or part, for multipart), ShannonStore mints a fresh 256-bit DEK at write time. The DEK encrypts the shard payloads; the *current ACTIVE KEK* for the relevant `keyId` wraps the DEK, and the KEK's version number is written into the envelope alongside the wrapped DEK. Only the wrapped DEK (+ version) is persisted — alongside the object's metadata in the API node's RocksDB.

A read pipeline:

1. Load the object's metadata, including the wrapped DEK, the KEK version it was wrapped under, and per-shard IVs.
2. Look up that exact `keyId`/version in the keystore and unwrap the DEK with it — there is no trial-and-error across multiple keys; the version is read directly from the first bytes of the envelope. A version whose state is `REVOKED` fails the unwrap with a `SecurityException`.
3. Decrypt each shard as it arrives from its data node.
4. Discard the unwrapped DEK after the last byte streams out.

The DEK never reaches the data node — every disk holds ciphertext only. A compromised data-node disk yields no readable bytes without the API node and the keystore.

### Key rotation

Rotation does **not** involve the master-key environment variable or a restart — it's a live, per-`keyId` operation on the leader:

- `rotateKey(keyId)` generates a brand-new random AES-256 KEK, marks the previous ACTIVE version `RETIRED` (not deleted — still needed to unwrap objects written under it), and installs the new key as `ACTIVE`. This runs on the leader and syncs to followers through the same replication channel as IAM.
- New objects (and any object whose DEK is re-issued through a rewrite path) wrap their DEK with the new ACTIVE KEK version. Existing objects keep their old wrapped DEK and are unwrapped using the exact version number embedded in their envelope — not by trying every known key.
- Admin REST surface (all require an admin JWT):

| Method | Path | Effect |
| --- | --- | --- |
| GET | `/admin/kms/list` | List all `keyId`s and their version states |
| GET | `/admin/kms/status/{keyId}` | Lifecycle detail for one `keyId` |
| POST | `/admin/kms/create` `{"keyId": "..."}` | Create a new `keyId` (first version `ACTIVE`) |
| POST | `/admin/kms/rotate/{keyId}` | Retire the current version, activate a new random one |

There is no comma-separated multi-key master-password mechanism — `SHANNONSTORE_MASTER_KEY` is validated as a single non-blank string, and rotating it is a separate, out-of-band re-keying of the keystore's own wrapping key rather than the normal `keyId` rotation path described above.

## What's encrypted

The same KMS pipeline protects every persistent surface the cluster owns:

| Surface | Encryption applied |
| --- | --- |
| Object shards on data-node disks | AES-256-GCM per shard, fresh IV per shard, key from the per-object DEK |
| Object metadata in API node RocksDB | each entry serialized then encrypted with the `default` KEK before `db.put()` |
| IAM state (`data/s3-metadata/iam/iam_state`) | encrypted with the `default` KEK on every `saveToLocalDb` — see [IAM](iam.md) |
| BucketManager state (versioning configs, ACLs, …) | same path as IAM, same `default` KEK |
| KMS-local RocksDB (the keystore blob itself) | encrypted with the PBKDF2-derived key from `SHANNONSTORE_MASTER_KEY`; protects the KEKs against a stolen disk |
| Cluster sync snapshots between API nodes | encrypted in transit *and* at rest after import |

Only ephemeral state — in-flight HTTP buffers, decoded part LRU cache, refresh-token map — is plaintext, and only in process memory.

## Compatibility paths

### Legacy plaintext upgrade

Clusters created before at-rest encryption was added carry a plaintext IAM/KMS blob. On startup the loader tries KMS-decrypt first; on failure it falls back to plaintext and immediately re-saves the blob encrypted. The log line

```
Legacy plaintext IAM detected on disk — re-saving as KMS-encrypted
```

is the one-time signal that the upgrade has just happened. Subsequent restarts find ciphertext on disk and take the normal path.

The fallback exists exactly once per blob — corruption that fails *both* paths is rethrown so a bad disk doesn't silently masquerade as a legacy plaintext blob.

### `aws kms` shape compatibility

Although ShannonStore's master key lives in an environment variable rather than AWS KMS, the object-level encryption headers and ETag rules match the AWS `aws:kms` server-side encryption shape closely enough that S3 SDKs configured for AWS KMS work against ShannonStore without code change.

## Operational guidance

- **Set `SHANNONSTORE_MASTER_KEY` from a secret manager** (HashiCorp Vault, AWS Secrets Manager, etc.), not from a static file checked into config. The cluster never persists it, but the process-launcher environment is the most common leak point.
- **Back up the master key out-of-band**. The keystore blob (and every KEK inside it) is useless without it. A cluster reinstall recovers nothing if the master key is also gone.
- **Rotate KEKs via `POST /admin/kms/rotate/{keyId}` after personnel changes or on a routine cadence** — it's a live, no-restart operation on the leader. The retired-but-not-deleted previous version costs nothing operationally; it just means new objects start under the new KEK version while old ones remain readable under theirs.
- **Audit on KMS RocksDB tampering**. The keystore is the only path between the master key and the object data. If a row is altered out of band, the AES-GCM tag verification fails on the next read.

## See also

- [Erasure Coding](ec.md) — the shard-level format that the per-object DEK encrypts.
- [Identity & Access Management](iam.md) — uses the same KEK to protect the IAM blob at rest.
- [Authentication & Authorization](auth-authz.md) — the JWT signing key is independent of the KEK; rotating one does not require rotating the other.
- [Backup & Restore](backup.md) — backup is taken from already-encrypted on-disk state, so an offsite copy is no easier to read than the live cluster.
