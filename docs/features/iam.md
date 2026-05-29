# Identity & Access Management (IAM)

ShannonStore ships with an AWS IAM-shaped access control system that gates every S3 and Admin REST request before the storage layer sees it. The model — users, groups, policies, access keys, STS temporary credentials — is intentionally familiar to anyone who's read an AWS IAM document, and the JSON policy schema is the AWS one.

A single `AuthManager` instance is the source of truth for the cluster:

```text
              ┌───────────────────────────────────────────┐
              │            AuthManager (per API)          │
              │                                           │
   usersById ─┤  user → User { groups[], credentials[] }  │
   groupsByName─►  group → IamGroup { policyNames[] }     │
   policiesByName ►  name → IamPolicy { Statement[] }     │
   userPolicies ─►  user → Set<policyName> (direct attach)│
   accessKeyIndex ►  AKID → User + AccessKeyRecord        │
   temporaryCredentials ►  AKID(ASIA) → STS creds         │
   refreshTokens ─►  refreshToken → SessionInfo           │
   revokedAccessTokens ─►  jti → expiry (logout tombstone)│
              │                                           │
              │  saveToLocalDb() ──► RocksDB (KMS-encrypted) │
              │  updateListener  ──► sync to peer API nodes  │
              └───────────────────────────────────────────┘
```

The full state is serialized as a single blob on every mutation, KMS-encrypted at rest in RocksDB at `data/iam-rocksdb/`, and broadcast to peers through the existing IAM sync channel so multi-API-node clusters converge without leader/follower divergence.

## Entities

### User

```java
class User {
    String userId;                       // unique id, doubles as the login name
    String password;                     // PBKDF2-hashed
    boolean requirePasswordChange;        // default-credential gate
    List<String> groups;                  // group names this user belongs to
    List<AccessKeyRecord> credentials;     // permanent + STS-minted keys
}
```

- `userId` is the natural identifier — group membership and access keys reference it.
- `password` is stored as a salted hash; raw passwords never appear at rest or in logs.
- `requirePasswordChange = true` blocks the user from issuing new tokens until they call the change-password endpoint with the real current password. The default `admin/admin` user is bootstrapped with this flag set, gating every privileged operation until it's cleared.

### Group

```java
class IamGroup {
    String groupName;
    List<String> policyNames;
}
```

Groups are pure policy attachment vehicles — they have no permissions of their own. A user inherits every policy on every group they belong to. The default install ships an `admin-group` carrying the `AdministratorAccess` policy and the `admin` user pre-joined to it.

### Policy

Policies are AWS-IAM-format JSON documents persisted by name:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadOnlyLake",
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": ["arn:aws:s3:::lake/*", "arn:aws:s3:::lake"]
    },
    {
      "Sid": "DenyPii",
      "Effect": "Deny",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::lake/pii/*"
    }
  ]
}
```

- `Action` and `Resource` accept a string or a list — wildcards `*` (zero-or-more) and `?` (one-char) are supported and compiled into bounded-cache Java `Pattern`s, so a complex policy reuses the same compiled regex across requests.
- `Effect` is `Allow` or `Deny`. Explicit `Deny` wins immediately — the evaluator short-circuits.
- `Sid` is optional, used purely for human auditability in admin tooling.
- `Version` is informational — only `Statement` is evaluated.

### Access keys (`AKIA…`)

Long-lived credentials for programmatic access, AWS-format access-key id (16 chars uppercase hex) + secret access key (32 chars hex):

```java
class AccessKeyRecord {
    String accessKey;     // AKID — used as the canonical lookup key
    String secretKey;     // shared secret for SigV4 / SigV2 signing
    long   createdAt;
    Long   expiresAt;     // null = no expiry
    String status;        // "Active" / "Inactive"
}
```

- Stored on the owning user; indexed in `accessKeyIndex` for O(1) S3 request validation.
- Inactive or expired keys fail `validateAccessKey()` and produce `401 InvalidAccessKeyId`.
- One user may carry several keys (typical operator pattern: hot key for the running workload, cold replacement minted ahead of rotation).

### STS temporary credentials (`ASIA…`)

Short-lived credentials minted by `AssumeRole`-style STS flow. Distinct prefix (`ASIA`) so the dispatch path treats them differently:

- Each temp credential is bound to a parent (long-lived) access key — the parent's authorizations are inherited.
- The S3 dispatch requires the matching `x-amz-security-token` on every request; missing or wrong token → 401.
- Lifetime is bounded by `shannonstore.api.sts.default.session.duration.seconds` (default per `ApiConfig`); the cleanup thread evicts expired entries every `cleanupIntervalMs`.
- Temp creds are excluded from on-disk persistence by default but are included in the sync snapshot so an STS session minted on one API node can authenticate on another (subject to clock-skew tolerance).

## Policy evaluation

The evaluator (`PolicyEvaluator`) walks the union of policies attached directly to the user and those attached to any of their groups. For each statement it tests `(Action, Resource)` against the requested pair using wildcard match.

```
            for each policy in (user_policies ∪ group_policies):
              for each Statement stmt:
                if matchAction(stmt.Action, requested) AND
                   matchResource(stmt.Resource, requestedArn):
                  if stmt.Effect == "Deny":  return DENY  ← short-circuit
                  if stmt.Effect == "Allow": allowed = true
            return ALLOW if allowed else ABSTAIN
```

Decision rules:

- `DENY` → `403 AccessDenied`, returned immediately even if other policies allow.
- `ALLOW` → request continues to the handler.
- `ABSTAIN` (no policy matched) → also denied, returned as `403 AccessDenied`. ShannonStore is deny-by-default per AWS spec.

Wildcard matching is **case-sensitive** — `s3:getobject` is not `s3:GetObject`. Match the SDK-emitted action verbs verbatim.

A bounded `PATTERN_CACHE` keeps compiled regexes around (default 10 000 entries, override via `shannonstore.api.iam.policy.pattern.cache.size`). On overflow the cache clears entirely rather than evicting LRU — cheaper, and the recompile happens at the rate of unique-pattern introduction, not per request.

## Action verbs

S3 actions follow standard AWS naming, derived from the HTTP verb × subresource combination at dispatch time. The verbs in active use today:

| Verb | Used by |
| --- | --- |
| `s3:ListAllMyBuckets` | `GET /` |
| `s3:ListBucket` | `GET /<bucket>` (object listing) |
| `s3:ListBucketVersions` | `GET /<bucket>?versions` |
| `s3:ListBucketMultipartUploads` | `GET /<bucket>?uploads` |
| `s3:ListMultipartUploadParts` | `GET /<bucket>/<key>?uploadId=…` |
| `s3:GetBucketLocation` | `GET /<bucket>?location` |
| `s3:GetBucketAcl` / `s3:GetObjectAcl` | `GET …?acl` |
| `s3:GetBucketVersioning` | `GET /<bucket>?versioning` |
| `s3:GetObject` | `GET /<bucket>/<key>` (also `HEAD`) |
| `s3:PutBucketVersioning` | `PUT /<bucket>?versioning` |
| `s3:PutBucketAcl` / `s3:PutObjectAcl` | `PUT …?acl` |
| `s3:PutObject` | `PUT /<bucket>/<key>`, CopyObject, multipart write paths |
| `s3:CreateBucket` | `PUT /<bucket>` |
| `s3:DeleteObject` | `DELETE /<bucket>/<key>`, `POST ?delete` (bulk) |
| `s3:DeleteBucket` | `DELETE /<bucket>` |
| `s3:AbortMultipartUpload` | `DELETE /<bucket>/<key>?uploadId=…` |

Build policies that reference exactly these verbs — actions Authors invent will silently fail the wildcard match and surface as `403 AccessDenied`.

## Resource ARNs

ARNs follow AWS S3 layout:

- Bucket: `arn:aws:s3:::<bucket>`
- Object: `arn:aws:s3:::<bucket>/<key>` (key path included literally — partition encodings stay encoded)

Wildcards work the way SDK authors expect: `arn:aws:s3:::*` matches every bucket and every key under them; `arn:aws:s3:::lake/etl/*` restricts to the `etl/` prefix on the `lake` bucket.

## Default install

The first start materializes a baseline so the cluster is usable immediately:

```text
policies:
  AdministratorAccess
    { Effect: Allow, Action: "*", Resource: "*" }

groups:
  admin-group
    policies: [AdministratorAccess]

users:
  admin / admin
    groups: [admin-group]
    requirePasswordChange: true        ← gate
```

Every privileged Admin REST and S3 path is blocked until the `admin` user changes the default password. See [Admin Password Recovery](admin-password-recovery.md) if the new password is lost.

## At-rest encryption + cluster sync

`AuthManager.saveToLocalDb()` writes a single blob to RocksDB at `data/iam-rocksdb/iam_state`. The blob is encrypted by the same `KmsProvider` that protects object data — losing the data-encryption key permanently locks the IAM state too.

On a successful write, the `updateListener` callback receives the encrypted snapshot and broadcasts it to peer API nodes through the standard IAM sync channel. Followers receive the bytes, decrypt locally, swap their in-memory maps under a write lock, and persist back to their own RocksDB. Loops are avoided because the listener guards on the local node's leadership state.

A legacy plaintext-blob path is kept for one upgrade window: if KMS-decrypt fails on load, the loader falls back to plaintext, re-encrypts, and writes back. The `Legacy plaintext IAM detected on disk — re-saving as KMS-encrypted` log line tells you the upgrade has just happened.

## Management surfaces

Day-to-day administration goes through the Admin UI (visual policy editor, user/group browser, access-key issuance) backed by the Admin REST endpoints on port `:8888`. The same REST endpoints are what cross-product tooling (chango, ontul, kiok, neorunbase) drives for automated provisioning.

Local recovery — when the `admin` user is locked out — uses a Unix domain socket on the API node, never an HTTP path. See [Admin Password Recovery](admin-password-recovery.md).

## See also

- [Authentication & Authorization](auth-authz.md) — token lifecycle, SigV4 internals, STS flow.
- [S3-Compatible API](s3-compatible-api.md) — full action × resource mapping for the dispatch table.
- [Encryption & Key Management](kms.md) — how the at-rest encryption that protects the IAM blob works.
- [Admin Password Recovery](admin-password-recovery.md) — the only path to reset the admin password when locked out.
