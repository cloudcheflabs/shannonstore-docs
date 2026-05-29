# Encryption & Key Management

ShannonStore encrypts every byte of object data and every byte of cluster metadata at rest. The implementation uses **envelope encryption** with AES-256-GCM throughout: a hierarchy of keys where the master key never directly touches data, the data-encryption keys never touch disk in plaintext, and only the bottom of the hierarchy fans out across the cluster.

```text
   ┌──────────────────────────────────────────┐
   │  Master Key (KEK)  ← env var, never on disk
   │   AES-256-GCM                            │
   └──────────────┬───────────────────────────┘
                  │  wraps
                  ▼
   ┌──────────────────────────────────────────┐
   │  Data Encryption Keys (DEK) — per object │
   │   AES-256-GCM, 96-bit IV per shard       │
   │   wrapped DEK persisted with object meta │
   └──────────────┬───────────────────────────┘
                  │  encrypts
                  ▼
   ┌──────────────────────────────────────────┐
   │  Object shards on disk (EC + compressed) │
   └──────────────────────────────────────────┘

   ┌──────────────────────────────────────────┐
   │  Metadata stores (IAM, BucketManager,    │
   │  KMS RocksDB) — encrypted with the same  │
   │  KEK on every write                       │
   └──────────────────────────────────────────┘
```

## Layers

### Master key (Key Encryption Key, KEK)

A 32-byte AES-256 key passed in through the environment:

```bash
export SHANNONSTORE_MASTER_KEY="…32 ASCII chars or longer…"
```

The KEK never leaves process memory after start. It is **not** written to RocksDB, not logged, not surfaced through any admin endpoint. Losing the value permanently locks every encrypted blob the cluster has on disk — object payloads, IAM state, the wrapped DEKs — all of them require this key to decrypt.

The master key is required at startup. The API node refuses to launch without it, so a misconfigured deployment fails closed rather than running with no encryption.

### Data Encryption Keys (DEK)

For each object (or part, for multipart), ShannonStore mints a fresh 256-bit DEK at write time. The DEK encrypts the shard payloads; the KEK encrypts the DEK. Only the wrapped DEK is persisted — alongside the object's metadata in the API node's RocksDB.

A read pipeline:

1. Load the object's metadata, including the wrapped DEK and per-shard IVs.
2. Unwrap the DEK with the in-memory KEK.
3. Decrypt each shard as it arrives from its data node.
4. Discard the unwrapped DEK after the last byte streams out.

The DEK never reaches the data node — every disk holds ciphertext only. A compromised data-node disk yields no readable bytes without the API node and KEK.

### Key rotation

Rotation does not re-encrypt existing objects (that would require rewriting petabytes). Instead the KEK is treated as the *current* wrapping key:

- New objects (and any object whose DEK is re-issued through a rewrite path) wrap their DEK with the new KEK.
- Existing objects keep their old wrapped DEK; the unwrap path holds *all* known KEKs in memory and tries each in turn until the AES-GCM authentication tag verifies.
- The set of known KEKs is the cluster's `KmsProvider` state — synced through the same replication channel as IAM. Lose the old KEK, lose access to objects written under it.

Operationally: rotate the KEK environment variable, restart API nodes one at a time, and let the cluster pick up the new key. The previous KEK must remain configured (as a comma-separated value) until every object encrypted under it has been re-wrapped or expired.

## What's encrypted

The same KMS pipeline protects every persistent surface the cluster owns:

| Surface | Encryption applied |
| --- | --- |
| Object shards on data-node disks | AES-256-GCM per shard, fresh IV per shard, key from the per-object DEK |
| Object metadata in API node RocksDB | each entry serialized then encrypted with KEK before `db.put()` |
| IAM state (`data/iam-rocksdb/iam_state`) | KEK-encrypted on every `saveToLocalDb` — see [IAM](iam.md) |
| BucketManager state (versioning configs, ACLs, …) | same path as IAM, same KEK |
| KMS-local RocksDB (wrapped DEK index) | KEK-encrypted; protects against a stolen data-node disk |
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
- **Back up the master key out-of-band**. The IAM/object blobs encrypted under it are useless without it. A cluster reinstall recovers nothing if the KEK is also gone.
- **Rotate annually or after personnel changes**. The dual-key window during rotation costs nothing operationally; it just means new objects start under the new KEK while old ones remain readable.
- **Audit on KMS RocksDB tampering**. The wrapped DEK index is the only path between the KEK and the object data. If a row is altered out of band, the AES-GCM tag verification fails on the next read — `SignatureFailure` in the API logs.

## See also

- [Erasure Coding](ec.md) — the shard-level format that the per-object DEK encrypts.
- [Identity & Access Management](iam.md) — uses the same KEK to protect the IAM blob at rest.
- [Authentication & Authorization](auth-authz.md) — the JWT signing key is independent of the KEK; rotating one does not require rotating the other.
- [Backup & Restore](backup.md) — backup is taken from already-encrypted on-disk state, so an offsite copy is no easier to read than the live cluster.
