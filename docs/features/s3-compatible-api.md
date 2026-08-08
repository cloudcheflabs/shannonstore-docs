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
| **PutObject** | `PUT /<bucket>/<key>` | Streaming upload via Netty NIO. Body is erasure-coded and KMS-encrypted before being persisted across data nodes. |
| **GetObject** | `GET /<bucket>/<key>` | Streaming download. Honors `Range` requests for partial / resumable reads. Sets `Content-Length`, `ETag`, `Last-Modified`. |
| **HeadObject** | `HEAD /<bucket>/<key>` | Returns the same metadata as GET without the body — used by SDKs for existence / size probes. |
| **DeleteObject** | `DELETE /<bucket>/<key>` | Replies `204 No Content` (S3 idempotent semantics — deleting a missing key still succeeds). |
| **CopyObject** | `PUT /<bucket>/<key>` with `x-amz-copy-source` | Server-side copy across buckets / keys. Does not stream through the client. |
| **DeleteObjects (Multi-object delete)** | `POST /<bucket>?delete` | Bulk delete payload in XML body — returns per-key `Deleted` and `Error` rows. |

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

Writes are leader-routed (`BUCKET_CONFIG_MUTATE`) so a `PUT` on one node is
immediately visible to a `GET` served by another node behind the proxy.

Still accepted as silent no-ops (default behaviour only): bucket/object `?acl` —
only the default owner ACL is modelled.

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

## Range requests

`GET /<bucket>/<key>` honors RFC 7233 `Range`:

```
Range: bytes=0-1023
Range: bytes=1024-
Range: bytes=-512        (last 512 bytes — suffix form)
```

A 206 Partial Content response carries the requested slice and a `Content-Range` header. Multi-range requests (`Range: bytes=0-99,200-299`) are not implemented — pick the single-range form.

Range requests are the path Iceberg/Parquet readers take to fetch column-chunk footers and only the columns they need, so partition-scan performance depends on them working correctly. The implementation caches recently-decoded EC parts in a bounded LRU so two consecutive range reads against the same object don't re-decode the same shard set.

## Versioning

Per-bucket versioning is toggled via `PUT /<bucket>?versioning` with the standard `<VersioningConfiguration><Status>Enabled</Status></VersioningConfiguration>` body. When enabled:

- PUT replaces the *current* version pointer; previous versions remain readable via `?versionId=…`.
- DELETE on a key without `versionId` writes a *delete marker* — a tombstone version. `ListObjects` then reports the key as missing; `ListObjectVersions` shows the marker.
- DELETE with `?versionId=…` permanently removes that specific version. The delete marker, if any, stays in place.
- Disabling versioning (`Suspended`) stops generating new version IDs but does not retroactively collapse history.

Versioning state is persisted in `BucketManager` and broadcast to peer API nodes through the IAM/bucket snapshot replication channel, so the per-bucket flag is consistent cluster-wide.

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
