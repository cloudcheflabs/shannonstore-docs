# Site Replication (multi-cluster active-active)

Site replication keeps two or more **separate ShannonStore clusters** in sync, the way
MinIO site replication does: every bucket and its objects written on any site are
replicated to all the other sites. It is the multi-cluster generalisation of
[Bucket Replication](bucket-configuration.md#bucket-replication) and is built on the
same leader-only worker.

```text
        write a.txt ──▶  Site A (cluster, :8080)  ◀───────────▶  Site B (cluster, :8080)  ◀── write b.txt
                              │  replicate all buckets             │  replicate all buckets
                              └──────────── a.txt ──▶──────────────┘
                              ◀────────────── b.txt ──────────────┘
        result: both sites hold a.txt AND b.txt
```

## Model

- **Active-active.** Each site is registered as the other's peer; writes flow both
  ways and converge. There is no primary.
- **Loop-safe by construction.** For every object the worker does a `HEAD` on the peer
  and only `PUT`s when the object is missing or its ETag differs. An object bouncing
  between two active-active sites therefore converges *without an infinite copy loop* —
  no wire tagging or oplog needed for the common case. (ShannonStore's ETag is the MD5
  of the plaintext, which is identical after a re-encrypting copy, so single-PUT
  objects match on both sides.)
- **Each site keeps its own KMS.** Objects are read **decrypted** at the source and
  re-PUT at the peer, which **re-encrypts with its own keys** — physically separated
  sites never share key material.
- **Leader-only.** The `SiteReplicationService` runs on the leader of each cluster, so
  each object is copied once per cycle.
- **Versioning** on the source bucket is recommended (and auto-enabled on the peer);
  every write is a new immutable version, which makes convergence a simple
  last-writer-wins on the current pointer.

## Configure peers

The config is cluster-global (one per cluster), leader-routed and replicated to every
node. Set it from the Admin UI (**Site Replication** page) or the admin REST API:

```bash
# On site A — register site B as a peer and enable
curl -X PUT http://siteA:8888/admin/replication/sites \
  -H "Authorization: Bearer <admin-jwt>" -d '{
    "enabled": true,
    "peers": [
      { "siteId": "B", "endpoint": "https://siteB:8080",
        "accessKey": "<B-access-key>", "secretKey": "<B-secret-key>", "region": "us-east-1" }
    ]
  }'

# On site B — register site A (the symmetric half makes it active-active)
curl -X PUT http://siteB:8888/admin/replication/sites \
  -H "Authorization: Bearer <admin-jwt>" -d '{
    "enabled": true,
    "peers": [ { "siteId": "A", "endpoint": "https://siteA:8080",
                 "accessKey": "<A-access-key>", "secretKey": "<A-secret-key>", "region": "us-east-1" } ]
  }'

curl     http://siteA:8888/admin/replication/sites/status  -H "Authorization: Bearer <admin-jwt>"
curl -X POST http://siteA:8888/admin/replication/sites/trigger -H "Authorization: Bearer <admin-jwt>"  # run a cycle now
```

The peer `endpoint` is the peer cluster's **S3 endpoint**, and the credentials are an
S3 access key/secret valid on the peer. The worker uses them to create buckets, set
versioning, and copy objects there.

```properties
# shannonstore.properties — reuses the bucket-replication cadence knob
shannonstore.api.replication.scan.interval.ms=60000   # cycle every 60s
```

## What is replicated (today)

- **Buckets** — created on the peer on demand (with versioning) so writes land.
- **Object data + content-type + tags** — current versions of every bucket.

A new cluster joins by being added as a peer on the existing sites (and adding them as
its peers); the next cycles copy the existing objects across, then steady-state only
new/changed objects move.

!!! note "Roadmap"
    Full control-plane replication (IAM users/keys/policies, all bucket sub-configs,
    KMS key metadata) and an incremental change-log with per-peer cursors are the next
    increments. Today's worker focuses on buckets + object data, which already covers
    cross-region DR and active-active object availability.

## See also

- [Bucket Configuration → Bucket Replication](bucket-configuration.md#bucket-replication) — single-bucket replication this builds on.
- [Encryption & Key Management](kms.md) — why each site re-encrypts with its own KMS.
- [Object Lock (WORM)](worm.md) and [Identity & Access Management](iam.md).
