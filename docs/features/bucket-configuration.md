# Bucket Configuration (Policy, CORS, Lifecycle, Tagging)

ShannonStore implements the S3 bucket-configuration subresources as **first-class,
persisted, cluster-consistent** features — not the silent compatibility stubs they
once were. Four configuration surfaces are supported, each with the canonical S3
wire format (so `aws s3api`, boto3, and the MinIO client work unchanged):

| Subresource | Purpose | Storage |
| --- | --- | --- |
| `?policy` | Resource-based **bucket policy** (JSON) — augments IAM and enables anonymous public access | raw JSON, per bucket |
| `?cors` | **CORS** rules (XML) — controls browser cross-origin access, incl. `OPTIONS` preflight | raw XML, per bucket |
| `?lifecycle` | **Object Lifecycle (ILM)** — typed `LifecyclePolicy`, expiration + abort-incomplete-MPU | typed model, per bucket |
| `?tagging` | **Tags** on a bucket and on individual objects | key/value map |

All four are owned by the cluster leader and replicated to followers — see
[Cluster consistency](#cluster-consistency-leader-routed-writes) below.

---

## Bucket Policy

A bucket policy is an AWS-style resource policy attached to a bucket:

```bash
aws --endpoint-url http://localhost:8000 s3api put-bucket-policy \
  --bucket lake --policy file://policy.json
aws --endpoint-url http://localhost:8000 s3api get-bucket-policy --bucket lake
aws --endpoint-url http://localhost:8000 s3api delete-bucket-policy --bucket lake
```

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicRead",
      "Effect": "Allow",
      "Principal": "*",
      "Action": ["s3:GetObject"],
      "Resource": ["arn:aws:s3:::lake/*"]
    }
  ]
}
```

### How the policy is evaluated

The policy is evaluated by `BucketPolicyEvaluator` **in addition to** the user's
IAM policies. The two combine with standard S3 semantics:

```text
                 IAM decision           Bucket-policy decision        Result
   authenticated  Allow        +        (any except Deny)        →    ALLOW
   authenticated  Deny/none    +        Allow                    →    ALLOW   (policy augments IAM)
   authenticated  any          +        explicit Deny            →    DENY    (explicit Deny always wins)
   anonymous      —            +        Allow to Principal "*"   →    ALLOW   (public access)
   anonymous      —            +        no matching Allow        →    401
```

Supported elements:

- **Effect** — `Allow` / `Deny`. An explicit `Deny` short-circuits and always wins.
- **Principal** — `"*"` (everyone, including anonymous) or `{ "AWS": "<id>" | [ ... ] }`.
  A bare access-key id or an ARN ending in `/<id>` / `:<id>` matches an authenticated caller.
- **Action** — `s3:*`, an exact action (`s3:GetObject`), or a wildcard suffix (`s3:Get*`).
- **Resource** — `arn:aws:s3:::bucket`, `arn:aws:s3:::bucket/*`, glob, or `*`.
- **Condition** — *conservatively skipped*: a statement carrying a `Condition` is
  treated as a non-match so it can never silently grant. Condition evaluation is a
  tracked future surface.

A malformed policy never grants — it evaluates to neutral.

### Anonymous public access

When a request arrives with **no `Authorization` header and no presigned query
signature**, ShannonStore evaluates the bucket policy for `Principal "*"`. If an
`Allow` matches the requested `(action, resource)`, the request proceeds
anonymously; otherwise it is rejected with `401`.

```bash
# After applying the PublicRead policy above:
curl http://localhost:8000/lake/report.csv        # 200, no credentials needed
curl http://localhost:8000/lake/private/secret     # 403/404 — not covered by the policy
```

Removing the policy immediately re-secures the bucket — the next anonymous GET
returns `401`.

---

## CORS

CORS rules let browser JavaScript from approved origins call the bucket directly.

```bash
aws --endpoint-url http://localhost:8000 s3api put-bucket-cors \
  --bucket assets --cors-configuration file://cors.json
aws --endpoint-url http://localhost:8000 s3api get-bucket-cors --bucket assets
aws --endpoint-url http://localhost:8000 s3api delete-bucket-cors --bucket assets
```

```json
{
  "CORSRules": [
    {
      "AllowedOrigins": ["https://app.example.com"],
      "AllowedMethods": ["GET", "PUT"],
      "AllowedHeaders": ["*"],
      "MaxAgeSeconds": 3000
    }
  ]
}
```

### Preflight (`OPTIONS`)

A browser preflight is an **unauthenticated** `OPTIONS` request. ShannonStore
dispatches it *before* SigV4 authentication (a preflight carries no credentials by
design), matches the `Origin` and `Access-Control-Request-Method` against the
bucket's CORS rules, and answers:

```bash
curl -i -X OPTIONS http://localhost:8000/assets/logo.png \
  -H "Origin: https://app.example.com" \
  -H "Access-Control-Request-Method: GET"
# HTTP/1.1 200 OK
# Access-Control-Allow-Origin: https://app.example.com
# Access-Control-Allow-Methods: GET, PUT
# Access-Control-Max-Age: 3000
```

A non-matching origin is rejected with `403` and no `Access-Control-Allow-*`
headers, exactly as a browser expects.

---

## Object Lifecycle (ILM)

Lifecycle rules express **time-based expiration** the same way AWS S3 and MinIO do.
The configuration is parsed into a typed `LifecyclePolicy` and persisted per bucket:

```bash
aws --endpoint-url http://localhost:8000 s3api put-bucket-lifecycle-configuration \
  --bucket staging --lifecycle-configuration file://lifecycle.json
aws --endpoint-url http://localhost:8000 s3api get-bucket-lifecycle-configuration --bucket staging
aws --endpoint-url http://localhost:8000 s3api delete-bucket-lifecycle --bucket staging
```

```json
{
  "Rules": [
    {
      "ID": "expire-tmp",
      "Filter": { "Prefix": "tmp/" },
      "Status": "Enabled",
      "Expiration": { "Days": 30 }
    }
  ]
}
```

Supported rule actions:

- **`Expiration.Days`** — delete objects older than N days under the rule's prefix.
- **`AbortIncompleteMultipartUpload.DaysAfterInitiation`** — reclaim parts of
  multipart uploads that were never completed.
- **`Filter.Prefix`** — restrict a rule to a key prefix. `Status` must be `Enabled`.

Transitions, storage-class, date-based and noncurrent-version rules are parsed but
not applied (ShannonStore has a single storage tier) — they never fail the sweep.

### The expiry scanner

Expiration is enforced by `LifecycleExpiryService`, a background scanner that:

- runs **only on the leader** API node (so each object is expired exactly once),
- is **disabled by default** and toggled at runtime from the Admin UI (Maintenance),
  mirroring the bitrot scrubber,
- **skips any object that is WORM-protected** — an active Object Lock retention or a
  legal hold always wins over a lifecycle rule (see [Object Lock (WORM)](worm.md)),
- sweeps on a configurable cadence:

```properties
# shannonstore.properties — §5c
# Interval in MILLISECONDS between lifecycle scan cycles (default 3600000 = 1 hour).
shannonstore.api.lifecycle.scan.interval.ms=3600000
```

---

## Tagging

Tags are user-defined key/value labels, supported on **both buckets and objects**:

```bash
# Bucket tags
aws --endpoint-url http://localhost:8000 s3api put-bucket-tagging --bucket lake \
  --tagging 'TagSet=[{Key=env,Value=prod},{Key=team,Value=data}]'
aws --endpoint-url http://localhost:8000 s3api get-bucket-tagging --bucket lake

# Object tags
aws --endpoint-url http://localhost:8000 s3api put-object-tagging --bucket lake \
  --key report.csv --tagging 'TagSet=[{Key=cost-center,Value=1234}]'
aws --endpoint-url http://localhost:8000 s3api get-object-tagging --bucket lake --key report.csv
```

Bucket tags live in the leader-owned bucket snapshot; object tags are stored on the
object's metadata and travel with it across the cluster.

---

## Cluster consistency (leader-routed writes)

Bucket configuration is **leader-owned state**, just like IAM and bucket ACLs. A
write that lands on any API node is applied authoritatively on the leader and
replicated to every follower, so a `PUT` on one node is immediately visible to a
`GET` served by another (behind the round-robin proxy):

```text
   client ──PUT ?policy──▶ API node B (follower)
                              │  internal NIO  (BUCKET_CONFIG_MUTATE_REQ)
                              ▼
                           API node A (leader)
                              │  apply to BucketManager
                              │  push bucket snapshot ──▶ all followers
                              ▼
   client ──GET ?policy──▶ API node C  →  sees the new policy
```

- The S3 data plane is custom NIO; only the leader mutates configuration and a
  follower **forwards** the mutation to the leader (`BUCKET_CONFIG_MUTATE_REQ`),
  mirroring the KMS-key mutation path.
- The admin REST plane (`/admin/browser/bucket-config/{bucket}/{type}`) is
  leader-forwarded for the same reason, and powers the Admin UI configuration panel.

This is why all four surfaces return consistent results regardless of which node a
request is routed to.

---

## Admin UI

Each of the four surfaces is editable from the **Browser → bucket → ⚙ (gear)**
configuration panel (administrators only), with tabs for Policy (JSON), CORS (XML),
Lifecycle (XML), and Tags (key/value editor). The panel talks to the
leader-forwarded admin endpoints listed above.

## See also

- [S3-Compatible API](s3-compatible-api.md) — the full operation dispatch table and authentication.
- [Object Lock (WORM)](worm.md) — retention / legal hold, which lifecycle expiration always defers to.
- [Identity & Access Management](iam.md) and [Authentication & Authorization](auth-authz.md).
