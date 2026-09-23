# S3-Compatible API

ShannonStore implements the Amazon S3 REST API at the wire level — the same canonical request signing, the same XML response shapes, the same HTTP status codes and error names — so applications built against S3 SDKs (boto3, aws-sdk-java, aws-cli, MinIO client, Iceberg/Spark/Trino S3FileIO, …) work with no code change. Only the endpoint URL switches.

```text
   Application (boto3 / SDK / aws-cli / Spark / Trino)
                       │
                       │  Authorization: AWS4-HMAC-SHA256  ─┐
                       ▼                                    │ identical
   ┌──────────────────────────────────────┐                 │ wire format
   │ ShannonStore API node (default :8080)│                 │ to AWS S3
   │  - SigV4 / V2 validator              │ ◀───────────────┘
   │  - dispatch table (verb × subresource)│
   │  - per-action IAM check (Allow/Deny) │
   │  - object plane → EC + KMS + cluster │
   └──────────────────────────────────────┘
```

## Supported operations

The dispatch table in `S3RequestHandler` routes the following verb × subresource combinations to fully implemented handlers. Each handler returns the canonical S3 XML response.

### Object plane

| Operation | Trigger | Notes |
| --- | --- | --- |
| **PutObject** | `PUT /<bucket>/<key>` | Streaming upload via Netty NIO. Body is encrypted &mdash; with a cluster key, or with a caller-supplied one under [SSE-C](#sse-c) &mdash; and then erasure-coded before being persisted across data nodes. Returns `x-amz-version-id` on a versioned bucket. |
| **GetObject** | `GET /<bucket>/<key>` | Streaming download. Honors `Range` requests for partial / resumable reads. Sets `Content-Length`, `ETag`, `Last-Modified`. |
| **HeadObject** | `HEAD /<bucket>/<key>` | Returns the same metadata as GET without the body — used by SDKs for existence / size probes. |
| **DeleteObject** | `DELETE /<bucket>/<key>` | Replies `204 No Content` (S3 idempotent semantics — deleting a missing key still succeeds). |
| **CopyObject** | `PUT /<bucket>/<key>` with `x-amz-copy-source` | Server-side copy across buckets / keys. Does not stream through the client. May carry a source key and a destination key at once, which is how an [SSE-C](#sse-c) object is re-keyed. |
| **DeleteObjects (Multi-object delete)** | `POST /<bucket>?delete` | Bulk delete payload in XML body — returns per-key `Deleted` and `Error` rows. |
| **PostObject (browser form upload)** | `POST /<bucket>` with `multipart/form-data` | The one S3 write a browser can make without an SDK. `${filename}` in the `key` field is expanded, and `success_action_status` chooses 200/201/204. The request is authorized the same way every other write is — the form's policy and signature fields are not evaluated, so an unsigned browser post is rejected earlier as unauthenticated. |
| **RestoreObject** | `POST /<bucket>/<key>?restore` | Brings a tiered object back to hot storage for a requested number of days. See [Storage Classes &amp; Tiering](storage-classes-tiering.md). |
| **GetObjectAttributes** | `GET /<bucket>/<key>?attributes` | `ETag`, `ObjectSize` and `StorageClass`, honouring `x-amz-object-attributes`. Fields this server does not compute are omitted rather than invented — a fabricated checksum would give a comparing client a false match. |
| **GetObjectAcl** | `GET /<bucket>/<key>?acl` | Synthesises an owner `FULL_CONTROL` grant; see the ACL note below. |
| **SelectObjectContent** | `POST /<bucket>/<key>?select&select-type=2` | SQL over a single object, CSV or JSON. See [S3 Select](#s3-select). |

### Multipart upload

| Operation | Trigger |
| --- | --- |
| **CreateMultipartUpload** | `POST /<bucket>/<key>?uploads` — returns an `UploadId`. |
| **UploadPart** | `PUT /<bucket>/<key>?uploadId=<id>&partNumber=N` — server caches part bytes; ETag is the part's MD5. |
| **UploadPartCopy** | `PUT /<bucket>/<key>?uploadId=<id>&partNumber=N` with `x-amz-copy-source` — copies a byte range from another object into a part. |
| **CompleteMultipartUpload** | `POST /<bucket>/<key>?uploadId=<id>` — assembles cached parts in part-number order and writes the final object. ETag becomes `MD5(concat MD5s) + "-" + partCount`. |
| **AbortMultipartUpload** | `DELETE /<bucket>/<key>?uploadId=<id>` — drops cached parts; reclaims their storage. |
| **ListParts** | `GET /<bucket>/<key>?uploadId=<id>` — returns every cached part in one response (`IsTruncated` is always `false`); `part-number-marker`-based pagination is not implemented. |
| **ListMultipartUploads** | `GET /<bucket>?uploads` |

The part-buffer store is the source of truth for in-flight uploads — it survives an API-node restart so a multipart begun on one node and resumed on another still completes (provided IAM state has propagated).

### Bucket plane

| Operation | Trigger | Notes |
| --- | --- | --- |
| **CreateBucket** | `PUT /<bucket>` | Owner becomes the caller. There is no existence/ownership check — a repeat `CreateBucket` call against a name that already exists (by the owner or anyone else) succeeds with `200` and resets the bucket's ACL rather than returning a conflict. |
| **DeleteBucket** | `DELETE /<bucket>` | Unlike AWS S3, the bucket does **not** need to be empty: a non-WORM-protected bucket is deleted along with every object it contains (WORM/object-lock-protected objects still block the delete). |
| **ListBuckets** | `GET /` | Returns every bucket the IAM credential can `s3:ListAllMyBuckets`. |
| **HeadBucket** | `HEAD /<bucket>` | Existence probe — used by `aws s3 ls` to discover buckets. |
| **ListObjects v1** | `GET /<bucket>?prefix=…&marker=…&max-keys=…` | `delimiter=/` returns `CommonPrefixes` for directory-style listing. |
| **ListObjects v2** | `GET /<bucket>?list-type=2&continuation-token=…&start-after=…` | Continuation token round-trips the next key (server-issued opaque base64 ≡ key for predictability). |
| **GetBucketLocation** | `GET /<bucket>?location` | Always returns an empty `<LocationConstraint></LocationConstraint>` (S3-legal shorthand for `us-east-1`) — no per-bucket or per-cluster region setting is consulted. |
| **GetBucketAcl** | `GET /<bucket>?acl` | Does **not** read the ACL persisted in `BucketManager`; it synthesizes a static `FULL_CONTROL` grant owned by the *requesting caller's* access key on every call. |
| **PutBucketVersioning** / **GetBucketVersioning** | `PUT/GET /<bucket>?versioning` | Versioning state is persisted per bucket and replicated in the IAM/bucket snapshot. |
| **ListObjectVersions** | `GET /<bucket>?versions` | Returns object versions interleaved with delete markers. |

### Bucket configuration subresources

The following subresources are **fully implemented, persisted, and replicated
across the cluster** — see [Bucket Configuration](bucket-configuration.md) for the
detailed behaviour, evaluation rules, and examples:

| Subresource | PUT | GET | DELETE | Notes |
| --- | --- | --- | --- | --- |
| `?policy` | persist JSON | return JSON / 404 `NoSuchBucketPolicy` | remove | Resource policy; augments IAM and enables anonymous public access (`Principal "*"`). |
| `?cors` | persist XML | return XML / 404 `NoSuchCORSConfiguration` | remove | Plus unauthenticated `OPTIONS` preflight handling. |
| `?lifecycle` | persist typed `LifecyclePolicy` | return XML / 404 `NoSuchLifecycleConfiguration` | remove | Leader-only expiry scanner; skips WORM-protected objects. |
| `?replication` | persist XML | return XML / 404 `ReplicationConfigurationNotFoundError` | remove | Leader-only async copy to a destination bucket/cluster. |
| `?tagging` (bucket **and** object) | persist `TagSet` | return `TagSet` | remove | Bucket tags in the leader snapshot; object tags on object metadata. |
| Object Lock `?object-lock` / `?retention` / `?legal-hold` | persist | return | — | See [Object Lock (WORM)](worm.md). |
| `?notification` | persist rules for this bucket | return this bucket's rules | — | Maps S3's per-bucket XML onto the cluster-wide rule set described in [Event Notifications](event-notifications.md). A destination naming no configured target is **refused**, not stored — such a rule silently drops every event it matches. Rules scoped `bucket="*"` are not returned here: they are not this bucket's configuration, and echoing them would invite a client to PUT them back narrowed to one bucket. |
| `?encryption` | persist XML | return XML / 404 `ServerSideEncryptionConfigurationNotFoundError` | remove | Stored and replicated as sent. |
| `?publicAccessBlock` | persist XML | return XML / 404 `NoSuchPublicAccessBlockConfiguration` | remove | Stored and replicated as sent. |
| `?ownershipControls` | persist XML | return XML / 404 `OwnershipControlsNotFoundError` | remove | Stored and replicated as sent. |
| `?website` | persist XML | return XML / 404 `NoSuchWebsiteConfiguration` | remove | Stored; this server does not serve static websites. |
| `?logging` | persist XML | return XML, or an empty `BucketLoggingStatus` when unset | — | 200-with-empty rather than 404, because logging-off is a valid state and SDKs read a 404 here as an error. |
| `?policyStatus` | — | `<IsPublic>` derived from the bucket policy | — | Computed from the policy on each call rather than stored: a stored answer and the policy can disagree, and the policy is what decides. |

The three "stored and replicated as sent" documents are kept as the XML the client
supplied rather than parsed into a model. Nothing acts on them today, and a parser
that dropped fields it did not understand would hand back something different from
what was stored — worse than not parsing, because a client reading its own
configuration back cannot tell a storage bug from a policy decision.

Writes are leader-routed (`BUCKET_CONFIG_MUTATE`) so a `PUT` on one node is
immediately visible to a `GET` served by another node behind the proxy.

### ACLs, and what is refused rather than ignored

Access control here is IAM plus the bucket policy. ACLs are a second, legacy
mechanism and are **not enforced**.

A `PUT ?acl` used to answer `200` while storing nothing. That is the failure mode
worth naming: a caller that *revoked* access was told it worked, and nothing
afterwards would reveal otherwise. So ACL requests that would change nothing are
still accepted — the canned `private`, an owner-only `FULL_CONTROL` grant — and
anything that would grant or remove access for someone else is refused with
`501 NotImplemented`, naming bucket policy as the mechanism that works.
`GET ?acl` still synthesises the owner `FULL_CONTROL` grant, which is consistent
with that model: the owner owns everything and there are no other grants.

The same principle applies to unimplemented sub-resources:

- **Unimplemented sub-resources** — `?accelerate`, `?requestPayment`,
  `?analytics`, `?inventory`, `?metrics`, `?intelligent-tiering`, `?torrent` —
  are refused with `501` before dispatch. They previously fell past every branch
  into the plain bucket handler, so `GET /bucket?accelerate` returned an *object
  listing* with a `200`. A wrong answer that parses is worse than an error.

## Authentication

Every S3 request must carry a valid AWS-style Authorization header. ShannonStore validates two formats:

### AWS Signature V4 (recommended — default in every modern SDK)

- Algorithm: `AWS4-HMAC-SHA256`.
- Validator reconstructs the canonical request from the wire-form path, query string, signed headers, and payload hash, then compares the recomputed signature with the one in the header.
- **Wire-form path matters**: SigV4 canonicalizes the URI exactly as it appears on the wire — including percent-encoding. The validator therefore uses `HttpRequest.rawPath()` (the undecoded path) rather than a `URLDecode`'d copy. This is the only way Iceberg- and Hive-style partition keys (`year%3D2026/month%3D05/file.parquet`) sign correctly under default SDK settings.
- Service names accepted: `s3` and `sts`.
- Streaming body (`aws-chunked`): payload hash is `STREAMING-AWS4-HMAC-SHA256-PAYLOAD`; the body is read through an `AwsChunkedInputStream` that strips chunk framing as data arrives. The `x-amz-decoded-content-length` header carries the real object size.

### Presigned URLs (query-string SigV4)

A presigned URL carries the entire SigV4 signature in the **query string** instead
of the `Authorization` header, so it can be handed to a browser, `curl`, or any
client with no credentials of its own:

```bash
url=$(aws --endpoint-url http://localhost:8000 s3 presign s3://lake/report.csv --expires-in 300)
curl "$url"        # 200 — auth is entirely in the X-Amz-* query parameters
```

When a request arrives with no `Authorization` header, the validator checks for
`X-Amz-Algorithm=AWS4-HMAC-SHA256` in the query string. If present, it extracts the
access key from `X-Amz-Credential`, reconstructs the canonical request from the
signed query parameters (`X-Amz-Date`, `X-Amz-Expires`, `X-Amz-SignedHeaders`,
`X-Amz-Signature`), and validates the signature and expiry exactly as for a header
signature. Anonymous-policy evaluation is only attempted for requests that are
*neither* header-signed *nor* presigned.

### AWS Signature V2 (legacy)

Older signing format kept for tooling that hasn't moved to SigV4. Same access-key index, same authorization decision afterwards.

### Authorization decision

After signature validation, the request is mapped to a `(action, resource-arn)` pair:

```text
PUT /lake/data.parquet            →  ("s3:PutObject", "arn:aws:s3:::lake/data.parquet")
GET /lake?list-type=2&prefix=etl/ →  ("s3:ListBucket", "arn:aws:s3:::lake")
DELETE /lake                      →  ("s3:DeleteBucket", "arn:aws:s3:::lake")
```

The pair is evaluated against the caller's effective IAM policies (user-attached + group-inherited). The evaluator is strict AWS semantics: explicit `Deny` wins over `Allow`, missing `Allow` denies by default. See [IAM](iam.md) and [Authentication & Authorization](auth-authz.md) for the full evaluator behaviour.

**Every sub-resource operation carries its own action.** A request that resolves
to no known operation maps to `s3:*`, which no ordinary policy grants — failing
closed, because an operation nobody mapped is an operation nobody decided the
permissions for.

| Request | Action |
| --- | --- |
| `PUT /b?policy` | `s3:PutBucketPolicy` |
| `DELETE /b?policy` | `s3:DeleteBucketPolicy` |
| `GET /b?policy` | `s3:GetBucketPolicy` |
| `PUT` / `DELETE /b?cors` | `s3:PutBucketCORS` |
| `PUT` / `DELETE /b?replication` | `s3:PutReplicationConfiguration` |
| `PUT` / `DELETE /b?lifecycle` | `s3:PutLifecycleConfiguration` |
| `PUT` / `DELETE /b?tagging` | `s3:PutBucketTagging` |
| `PUT` / `DELETE /b?notification` | `s3:PutBucketNotification` |
| `PUT` / `DELETE /b?encryption` | `s3:PutEncryptionConfiguration` |
| `GET /b?policyStatus` | `s3:GetBucketPolicyStatus` |
| `GET /b/k?attributes` | `s3:GetObjectAttributes` |
| `POST /b/k?restore` | `s3:RestoreObject` |

!!! warning "This was wrong before, and the wrong direction was privilege escalation"
    Only a handful of sub-resources were mapped; everything else fell through to
    the bucket-level default. `PUT /b?policy` was therefore guarded by
    `s3:CreateBucket` and `DELETE /b?cors` by `s3:DeleteBucket` — so a caller
    holding `s3:DeleteBucket` could rewrite or remove a bucket's access-control
    policy, and a caller holding exactly `s3:PutBucketPolicy` could not set one.

    Sub-resources are also matched by parsed query **key** now, not by
    `uri.contains("?policy")` — which is also true of `?policyStatus`, so a
    substring probe could route one operation into another's handler and another
    operation's permission check.

## ETag semantics

ShannonStore preserves the exact ETag rules that AWS S3 SDKs depend on for client-side verification:

| Path | ETag format |
| --- | --- |
| Single-PUT object | lowercase hex of `MD5(plaintext body)` |
| Multipart-completed object | lowercase hex of `MD5(concat(each part's raw MD5 bytes)) + "-" + partCount` |
| HeadObject / GetObject responses | the persisted ETag, surrounded by double quotes per S3 wire format |
| Pre-existing legacy objects | empty when written before the field existed; SDK still works because the ETag is optional |

So `aws s3 cp --validate` and `boto3.upload_file` integrity checks pass without any client-side workaround.

## Content-MD5 enforcement

When the client opts into stronger write-time verification:

```
Content-MD5: <base64(md5(body))>
```

the server recomputes MD5 of the received bytes and rejects mismatches with `400 BadDigest`. This applies to both PutObject and UploadPart. The check requires materializing the body to a buffer — clients that need streaming throughput should omit Content-MD5 and rely on TCP / TLS integrity plus ETag verification post-write.

### Additional checksums

The AWS additional-checksum headers are verified the same way and echoed back on
the response:

```
x-amz-checksum-crc32
x-amz-checksum-crc32c
x-amz-checksum-sha1
x-amz-checksum-sha256
```

Newer AWS SDKs compute one of these by default and send it on every upload,
usually with no `Content-MD5` at all. Their presence therefore forces the same
buffered path Content-MD5 does: a checksum header is a promise the server is
expected to check, and checking it requires the whole body. Ignoring one would be
worse than not supporting checksums — a corrupted body would be stored, reported
as a success, and the client would believe it had verified the write.

## Conditional requests

`GET` and `HEAD` both honour RFC 7232 pre-conditions:

| Header | Satisfied | Not satisfied |
| --- | --- | --- |
| `If-Match` | request proceeds | `412 PreconditionFailed` |
| `If-None-Match` | request proceeds | `304 Not Modified` (with `ETag` and `Last-Modified`) |
| `If-Unmodified-Since` | request proceeds | `412 PreconditionFailed` |
| `If-Modified-Since` | request proceeds | `304 Not Modified` |

Evaluation follows the RFC's order — `If-Match`, then `If-Unmodified-Since`, then
`If-None-Match`, then `If-Modified-Since` — and a date condition is ignored when
its stronger ETag counterpart is present on the same request, so a client sending
both does not get a `412` it did not ask for.

`Last-Modified` is compared at second resolution, matching what goes on the wire;
comparing finer would report a sub-second difference as "modified".

!!! note "This used to differ between GET and HEAD"
    HEAD honoured these headers and GET ignored them entirely, so a conditional
    GET issued for caching re-sent the whole body every time, and `If-Match` gave
    a reader no protection on the one verb that returns data. Both now share one
    implementation.

## Range requests

`GET /<bucket>/<key>` honors RFC 7233 `Range`:

```
Range: bytes=0-1023
Range: bytes=1024-
Range: bytes=-512        (last 512 bytes — suffix form)
```

A 206 Partial Content response carries the requested slice and a `Content-Range` header.

**Multi-range** (`Range: bytes=0-99,200-299`) is answered as `multipart/byteranges`,
one part per range with its own `Content-Range`. Because each part is buffered,
a multi-range request whose parts add up to more than
`shannonstore.api.s3.small.object.threshold` is served as the whole object
instead — a client asking for most of an object in slices is better served the
object than by having the server hold it all in memory, and RFC 7233 permits
ignoring the ranges.

Two edge cases behave as the RFC requires rather than as errors:

- A range entirely past the end of the object gets `416` with
  `Content-Range: bytes */<length>`, so the client learns the real length without
  a second request.
- A **malformed** `Range` is ignored and the whole object returned. It used to be
  a `500`: the header was split on `-`, so `bytes=0-99,200-299` fed `"99,200"` to
  the number parser.

Range requests are the path Iceberg/Parquet readers take to fetch column-chunk footers and only the columns they need, so partition-scan performance depends on them working correctly. The implementation caches recently-decoded EC parts in a bounded LRU so two consecutive range reads against the same object don't re-decode the same shard set.

## Server-side encryption

Three mechanisms, differing in who holds the key.

| | Who holds the key | Request headers | Needed to read |
|---|---|---|---|
| **Cluster default** | The cluster | none | nothing |
| **SSE-S3** | The cluster | `x-amz-server-side-encryption: AES256` | nothing |
| **SSE-KMS** | The cluster | `x-amz-server-side-encryption: aws:kms` + `…-aws-kms-key-id` | nothing |
| **SSE-C** | **The caller** | `x-amz-server-side-encryption-customer-{algorithm,key,key-md5}` | the same key, on every request |

`GET`, `HEAD` and `PUT` report back which of these an object actually got, so a
client that asked for encryption can confirm it happened. An object encrypted
only because the cluster encrypts everything reports nothing — that is not
something the caller asked for, and S3 does not report it either.

### SSE-C

The key is used for one request and dropped; it is never stored. What is kept
beside the object is a salted HMAC of it, which answers *is this the same key*
and nothing else. It is salted so that two objects written with one key do not
carry the same value, which would otherwise let anyone reading metadata group
objects by key.

Supported on `PUT`, `GET`, `HEAD`, `UploadPart`, `CopyObject`, `UploadPartCopy`
and `SelectObjectContent`. A copy may carry two keys at once — the source key in
`x-amz-copy-source-server-side-encryption-customer-*` and the destination key in
the usual headers — which is how an object is re-keyed, or decrypted into a
plain one.

Three refusals, and the difference between them is actionable:

| Situation | Status | What to do |
|---|---|---|
| SSE-C object, no key sent | `400 InvalidRequest` | resend with the key |
| SSE-C object, wrong key | `403 AccessDenied` | retrying will not help |
| Key sent for an object that is not SSE-C | `400 InvalidRequest` | drop the headers |

The third is a refusal rather than a shrug on purpose. Ignoring the key would
hand back plaintext to a caller who believes the bytes were protected by a key
only they hold.

`ListObjects`, `ListObjectVersions` and `DeleteObject` do **not** need the key:
only the bytes are encrypted, and an operator who has lost a key must still be
able to remove the object.

### What SSE-C cannot do

Anything that reads an object without a request behind it cannot open an SSE-C
object, because there is no key to be had. These paths skip it and say so rather
than failing halfway or writing bytes nobody can read:

- **Replication** and **site replication** skip the object and record
  `SSE_C_SKIPPED` against it. Configuring a replication rule on a bucket that
  already holds SSE-C objects is still accepted — refusing would strand every
  other object in the bucket — but logs a warning at that moment, so this is
  not discovered later. See
  `shannonstore.api.replication.ssec.scan.limit` in
  [Configuration](configuration.md).
- **Tiering** leaves the object in place.
- The **admin console** answers `409` on a download rather than failing as a
  server error.

Bitrot scrubbing and EC repair are unaffected: they work on ciphertext shards
and their checksums, and never need the plaintext.

## Versioning

Per-bucket versioning is toggled via `PUT /<bucket>?versioning` with the standard `<VersioningConfiguration><Status>Enabled</Status></VersioningConfiguration>` body. When enabled:

- PUT replaces the *current* version pointer; previous versions remain readable via `?versionId=…`.
- DELETE on a key without `versionId` writes a *delete marker* — a tombstone version. `ListObjects` then reports the key as missing; `ListObjectVersions` shows the marker.
- DELETE with `?versionId=…` permanently removes that specific version. The delete marker, if any, stays in place.
- Disabling versioning (`Suspended`) stops generating new version IDs but does not retroactively collapse history.

Versioning state is persisted in `BucketManager` and broadcast to peer API nodes through the IAM/bucket snapshot replication channel, so the per-bucket flag is consistent cluster-wide.

`PUT` returns `x-amz-version-id` for the version it created, so a client can
address what it just wrote instead of listing versions and guessing which one
is its own.

Every version keeps its own bytes, including under successive writes to the same
key with no pause between them. That is worth stating explicitly because writes
land in a part buffer and are committed to their final location asynchronously:
the commit has to find the version it belongs to, not whichever version is
newest by the time it runs. A version only becomes the current one if it is
genuinely newer, so a late commit of an older version cannot roll the object
back while the version listing still names a newer one as latest.

## Error response shape

Errors follow the canonical S3 XML:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Error>
  <Code>NoSuchKey</Code>
  <Message>The specified key does not exist.</Message>
  <Resource>/lake/missing.parquet</Resource>
  <RequestId>…</RequestId>
</Error>
```

Most-encountered codes:

| Status | Code | Cause |
| --- | --- | --- |
| 400 | `InvalidRequest` | Malformed body, missing required parameter |
| 400 | `BadDigest` | `Content-MD5` mismatched the received body |
| 401 | `Unauthorized` | No `Authorization` header |
| 401 | `InvalidAccessKeyId` | Access key not in IAM index (after one cluster reload retry) |
| 403 | `SignatureDoesNotMatch` | SigV4 / V2 verification failed |
| 403 | `AccessDenied` | IAM policy did not allow the `(action, resource)` pair |
| 404 | `NoSuchBucket` | Bucket does not exist |
| 404 | `NoSuchKey` | Object does not exist |
| 404 | `NoSuchBucketPolicy` / `NoSuchCORSConfiguration` / `NoSuchLifecycleConfiguration` | Subresource never configured |

The body is intentionally identical to AWS S3 for the codes above — SDK error parsers don't need a ShannonStore-specific path. Two AWS-standard codes are **not** currently emitted: `CreateBucket` never returns `409 BucketAlreadyOwnedByYou` (see the CreateBucket row above — it always succeeds), and [Maintenance Mode](maintenance.md)'s `503` response carries no XML `<Error>` body or code at all (just `Retry-After: 30`), not `SlowDown`.

## S3 Select

`POST /<bucket>/<key>?select&select-type=2` runs SQL over a single object and
returns an AWS event stream.

### Formats

| | Supported | Refused |
| --- | --- | --- |
| **Input** | CSV, JSON (`Type=LINES` and `Type=DOCUMENT`) | **Parquet** and any `CompressionType` other than `NONE` — `501 NotImplemented` |
| **Output** | CSV, JSON | — |

XML is not a Select format — it is not one in AWS either, on input or output.

A `DOCUMENT` object is parsed as one JSON value; an array yields a record per
element. Reading it the way `LINES` is read would hand the parser fragments of a
value.

### The accepted SQL

```sql
SELECT *  |  COUNT(*)  |  col [, col ...]
FROM S3Object [alias]
[WHERE <condition>]
[LIMIT n]
```

A condition is comparisons (`=`, `!=`, `<>`, `<`, `<=`, `>`, `>=`) and `LIKE`,
combined with `AND` / `OR` / `NOT` and parentheses. Columns are addressed by name
(CSV header or JSON field) or by position (`s._1`). There are **no functions, no
arithmetic, no casts, no subqueries and no joins.**

Anything outside that grammar is rejected with `400 InvalidExpression` **before a
single byte of the object is read**.

### Why it is this small

S3 Select is a server parsing caller-supplied text and evaluating it over
caller-supplied data. That is the shape of a long line of CVEs in this exact
feature elsewhere, so the scope is a deliberate choice rather than an unfinished
one:

- The grammar is hand-written and rejects by default. A general SQL engine would
  import all of that surface to support a language whose useful form here is one
  source, no joins, no subqueries.
- `LIKE` compiles to a linear matcher, not a regular expression. A pattern built
  from caller input and handed to a regex engine is its own denial-of-service
  class.
- **Parquet input is refused.** A binary parser over caller-supplied bytes is the
  risk itself, and the engines that read Parquet here — Spark, Trino, Iceberg —
  do not use Select at all: they read footers and column chunks with
  [ranged GETs](#range-requests), which this server serves.
- Every scan is bounded. See the settings below.

### Limits

| Property | Default | Bounds |
| --- | --- | --- |
| `shannonstore.api.s3.select.max.object.bytes` | `134217728` (128 MiB) | Largest object a Select will scan. A Select reads the whole object into memory; a larger object is refused with `ObjectTooLarge` rather than attempted. |
| `shannonstore.api.s3.select.max.record.bytes` | `1048576` (1 MiB) | Longest single record. Keeps a file with no line breaks — a truncated upload, a generated one-line file — from turning one request into an allocation the size of the object. |
| `shannonstore.api.s3.select.max.fields` | `4096` | Most fields on one record. A line of nothing but delimiters would otherwise produce one list entry per byte. |

### Response

An AWS event stream: `Records` messages carrying the result, then `Stats`, then
`End`. Because the stream commits the HTTP status as `200` before the scan runs,
a failure discovered mid-scan is delivered as an in-band error event — a stream
that simply stops is indistinguishable from a dropped connection.

```bash
aws s3api select-object-content \
  --bucket lake --key sales.csv \
  --expression "SELECT name FROM S3Object s WHERE s.qty > 100" \
  --expression-type SQL \
  --input-serialization '{"CSV":{"FileHeaderInfo":"USE"}}' \
  --output-serialization '{"CSV":{}}' \
  /dev/stdout
```

## Two-port topology

Object storage and admin/management traffic are bound to separate sockets so the dataplane and the controlplane never compete:

| Port (default) | Surface | Audience |
| --- | --- | --- |
| **8080** | S3 REST | applications, SDKs, AWS CLI |
| **8888** | Admin REST + Admin UI | operators, IAM management, cross-product tooling |

The admin port is the only place CCL cross-product tooling (chango, ontul, kiok, neorunbase) talks to ShannonStore — for IAM bootstrap, access-key minting, and configuration sweeps. It must never face the public internet; front it with the same nginx that fronts the data port (see [Nginx Reverse Proxy](../operations/nginx-proxy.md)).

## See also

- [Authentication & Authorization](auth-authz.md) — SigV4 canonical request internals, JWT session lifecycle, STS flow.
- [IAM](iam.md) — policy schema, evaluator semantics, action / resource ARNs.
- [Distributed Architecture](distributed-architecture.md) — how object writes fan out to data nodes.
- [Erasure Coding](ec.md) — the storage layout PUT and GET operate against.
- [Maintenance Mode](maintenance.md) — the 503 + `Retry-After` source.
