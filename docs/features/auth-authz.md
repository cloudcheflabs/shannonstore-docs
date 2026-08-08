# Authentication & Authorization

ShannonStore runs two authentication chains side by side — one for the AWS S3 data plane on port `:8080`, one for the Admin REST and Admin UI on port `:8888`. They share the same IAM state (users, groups, policies, access keys) so a credential issued through the Admin REST is immediately usable for S3 calls, but the wire formats are different and the chains are evaluated in different places.

```text
   ┌───────────────────────────────────┐    ┌───────────────────────────────────┐
   │  S3 API   :8080                   │    │  Admin REST + Admin UI   :8888    │
   │                                   │    │                                   │
   │  Authorization: AWS4-HMAC-SHA256… │    │  Authorization: Bearer <JWT>      │
   │  └─ SigV4 canonical-request check │    │  └─ JWT signature + revocation    │
   │                                   │    │                                   │
   │  Authorization: AWS <ak>:<sig>    │    │  Cookie session                   │
   │  └─ SigV2 (legacy SDKs)           │    │                                   │
   │                                   │    │  Username + password (login form) │
   │  x-amz-security-token (STS)       │    │                                   │
   │  └─ ASIA temp creds               │    │                                   │
   └───────────────────────────────────┘    └───────────────────────────────────┘
                       │                                       │
                       └────────── shared AuthManager ─────────┘
                              (users, groups, policies,
                               access keys, sessions)
```

## Admin UI / API authentication

### Login

`POST /admin/auth/login` against the Admin REST exchanges a username + password for a short-lived JWT (the *access token*) and a longer-lived refresh token. The access token carries the user id as `sub`, an `isAdmin` boolean claim, the `requirePasswordChange` flag, and a random JWT id (`jti`) used for explicit revocation.

```json
{
  "token":         "<short-lived JWT, 15 min default>",
  "refreshToken":  "<opaque UUID, 24 h default>",
  "userId":        "admin",
  "expiresAt":     1779999999999,
  "isAdmin":       true,
  "requirePasswordChange": false
}
```

The HMAC-SHA256 signing key is configurable via `setJwtSecret(...)` — minimum 32 bytes. Default expirations are overridable via `setJwtExpirations(access, refresh, cleanup)`:

| Token | Default | Purpose |
| --- | --- | --- |
| Access token (JWT) | 15 min | Carried as `Authorization: Bearer …` on every admin REST call. |
| Refresh token (UUID) | 24 h | Exchanged at `/admin/auth/refresh` to mint a new access token without re-entering the password. |
| Cleanup interval | 3 h | Daemon thread evicting revoked-tokens and expired refresh/STS entries. |

### Refresh

`POST /admin/auth/refresh` with `{"refreshToken": "…"}` returns the same JSON shape as login. The refresh token's stored `SessionInfo` carries the userId so even after the access token has expired the new one is minted for the right principal. Refresh tokens themselves expire when their `SessionInfo` does — re-login is required after that.

### Logout

`POST /admin/auth/logout` parses the bearer JWT, extracts `jti`, and inserts the id into `revokedAccessTokens` keyed against the token's expiration timestamp. Subsequent `validateSession()` calls reject the token instantly even if its signature is still valid. The cleanup thread purges revoked entries once their original expiry passes, so the map size grows with active sessions only.

### Session validation

Every admin REST call below the auth gate runs through `validateSession()`:

1. Parse + verify the JWT against the signing key. Mismatched signature → 401.
2. Look up `jti` in `revokedAccessTokens`. Hit → 401.
3. Look up `sub` (user id) in the in-memory `usersById` map. Missing → 401.
4. Stamp the response context with `isAdmin` and `requirePasswordChange` for downstream checks.

The handler then enforces the admin-only gate on management endpoints (user/group/policy mutation, access-key issuance) and, in addition, blocks any request from a user whose `requirePasswordChange` flag is still true.

### Default-credential gate

The very first start materializes `admin / admin` with `requirePasswordChange = true`. While the flag is set, **every privileged path returns 403** — admin UI navigation, IAM mutation, access-key issuance, even the S3 control plane through the admin REST. The only callable endpoint is `POST /admin/auth/change-password`, which clears the flag once the caller proves they know the current password and supplies a new one that differs from it.

See [Admin Password Recovery](admin-password-recovery.md) for the out-of-band recovery channel when the admin password is forgotten.

## S3 API authentication

The S3 plane on `:8080` accepts the standard AWS Authorization formats with the standard AWS canonical request rules. Two formats are validated:

### AWS Signature V4 (`AWS4-HMAC-SHA256`)

Recommended — what every modern SDK emits. The validator (`AwsSigV4Validator`) re-derives the signature server-side and compares with the one in the header.

```text
                  Canonical request

   HTTPMethod \n
   CanonicalURI       ← raw, wire-form, not URL-decoded
   CanonicalQueryString ← sorted by name+value, kept percent-encoded
   CanonicalHeaders  ← lowercase name + trimmed value, '\n'-terminated
   SignedHeaders      ← sorted, ';' joined
   PayloadHash        ← SHA-256 hex of the body or UNSIGNED-PAYLOAD
                      or STREAMING-AWS4-HMAC-SHA256-PAYLOAD for chunked
```

Every choice ShannonStore makes here matches AWS:

- **Service names accepted**: `s3` and `sts`.
- **CanonicalURI** is read from `HttpRequest.rawPath()` — never URL-decoded. This is the difference that lets Iceberg-/Hive-style partition keys (`year%3D2026/month%3D05/file.parquet`) sign correctly under default SDK settings. A previous bug where the path was decoded was caught by `AwsSigV4ValidatorTest.partitionedKeyFails_withDecodedPath`, which now permanently pins the rule.
- **CanonicalQueryString** preserves the SDK's wire encoding; the server does not re-encode. Re-encoding would double-encode anything the SDK already escaped, breaking the signature. Keys with no value (e.g. `?uploads`) get a trailing `=` per the AWS canonicalization rule.
- **Payload hash** is `SHA-256 hex(body)` for normal PUTs, the literal `UNSIGNED-PAYLOAD` when the header opts out, and `STREAMING-AWS4-HMAC-SHA256-PAYLOAD` for chunked uploads — the server reads the value from `x-amz-content-sha256` and trusts the header for the streaming variant.
- **Signing key derivation**: `kDate = HMAC(AWS4+secret, date) → kRegion → kService → kSigning = HMAC(prev, "aws4_request")`. Standard AWS chain — kept verbatim so the SDK-side and server-side keys are identical given the same inputs.

When validation fails, the failure mode is intentionally verbose in the master log (canonical request, string-to-sign, signed-headers list, computed vs. provided signature). Operators can reproduce the SDK's expected canonical request by hand and diff against the dump.

### AWS Signature V2 (legacy)

Older format: `Authorization: AWS <accessKey>:<base64-signature>`. Kept for tools that haven't migrated (some older Hadoop S3A configurations, certain MinIO mirrors). Same access-key lookup, same authorization decision afterwards.

### Access-key lifecycle

On every S3 request the validator:

1. Parses the access key id from the `Authorization` header (`Credential=AKID/...` or `AWS AKID:...`).
2. Looks up the AKID in `accessKeyIndex`. Miss → triggers a cluster-wide IAM reload (in case the key was minted on a peer node), retries once. Persistent miss → `401 InvalidAccessKeyId`.
3. For temporary credentials (`ASIA…` prefix), additionally requires `x-amz-security-token` matching the stored token.
4. Loads the user's hashed secret and verifies the SigV4/V2 signature. Mismatch → `403 SignatureDoesNotMatch`.

A successful validation hands the principal id to the authorization phase.

## STS temporary credentials

ShannonStore implements an `AssumeRole`-style STS flow on the S3 endpoint at `POST /` with a form-encoded body — the AWS canonical wire shape:

```
POST / HTTP/1.1
Host: shannonstore.local:8080
Content-Type: application/x-www-form-urlencoded
Authorization: AWS4-HMAC-SHA256 … (parent credential signs the call)

Action=AssumeRole&Version=2011-06-15&RoleSessionName=etl-shard-12&DurationSeconds=3600
```

The handler mints a new `AccessKeyRecord` with the `ASIA` prefix, returns the AWS canonical XML response:

```xml
<AssumeRoleResponse xmlns="https://sts.amazonaws.com/doc/2011-06-15/">
  <AssumeRoleResult>
    <AssumedRoleUser>
      <Arn>arn:aws:iam::000000000000:role/default/etl-shard-12</Arn>
      <AssumedRoleId>AROAXXXXXXXXXXXXXXXXX:etl-shard-12</AssumedRoleId>
    </AssumedRoleUser>
    <Credentials>
      <AccessKeyId>ASIAXXXXXXXXXXXXXXXX</AccessKeyId>
      <SecretAccessKey>…</SecretAccessKey>
      <SessionToken>…</SessionToken>
      <Expiration>2026-05-30T01:23:45Z</Expiration>
    </Credentials>
    <PackedPolicySize>1</PackedPolicySize>
  </AssumeRoleResult>
  <ResponseMetadata>
    <RequestId>…</RequestId>
  </ResponseMetadata>
</AssumeRoleResponse>
```

Lifetime rules:

- `DurationSeconds` defaults to `shannonstore.api.sts.default.session.duration.seconds` (3600) when omitted, and is always clamped to the AWS-compatible range `[900, 43200]` seconds regardless of what the client requests.
- The temp credential is bound to the *parent* access key — every S3 call must carry the security token and the parent's authorizations propagate.
- A cleanup thread evicts expired temp credentials every `cleanupIntervalMs`.
- Temp creds are excluded from regular at-rest persistence by default but ride the IAM sync snapshot so other API nodes see them quickly.

The STS path is pre-synced — the credentials are pushed to peer API nodes *before* the response goes back to the client. This avoids the race where a worker hands the freshly issued ASIA to a co-process that hits a different API node before the credential has propagated.

## S3 authorization flow

Authentication establishes *who*. Authorization decides *what*. Every authenticated S3 request is mapped to an `(Action, ResourceArn)` pair using the URI dispatch table:

```text
   PUT  /lake/data.parquet                  →  s3:PutObject       arn:aws:s3:::lake/data.parquet
   PUT  /lake?versioning                   →  s3:PutBucketVersioning arn:aws:s3:::lake
   GET  /lake?list-type=2&prefix=etl/      →  s3:ListBucket      arn:aws:s3:::lake
   POST /lake?delete                       →  s3:DeleteObject    arn:aws:s3:::lake
   DELETE /lake/data.parquet               →  s3:DeleteObject    arn:aws:s3:::lake/data.parquet
   DELETE /lake/x?uploadId=abc             →  s3:AbortMultipartUpload arn:aws:s3:::lake/x
```

The pair is evaluated by `PolicyEvaluator.evaluate()` against the union of policies attached to the caller's user and groups. Decision rules:

- **Explicit `Deny` short-circuits** — the first `Deny` whose `Action` and `Resource` match returns `403 AccessDenied` immediately, regardless of how many `Allow` statements precede or follow it.
- An `Allow` is required for the request to proceed — there is no implicit allow even for the bucket owner.
- No matching statement at all → `ABSTAIN` → also `403 AccessDenied`. Deny-by-default.
- Wildcards in `Action` (`s3:*`, `s3:Get*`) and `Resource` (`arn:aws:s3:::lake/*`) are supported and compiled to bounded-cache regex.

See [IAM](iam.md) for the entity model, the verb list, and the at-rest encryption / cluster sync behaviour the policy state rides on.

## Cross-product login wire compatibility

The Admin REST login response carries both the legacy ShannonStore keys (used by the Admin UI) and the cross-product convention keys other CCL products (ontul, chango, kiok, neorunbase) consume. The same JWT appears under both `token` and `accessToken`; the user id appears under both `userId` and `username`. Either set of keys is acceptable in the request body (`userId` *or* `username` for the login form). This makes ShannonStore's auth surface a superset of the AWS-style shape every CCL tool already understands, so cross-product credential sweepers (e.g. chango's admin-credentials-sweeper) drive ShannonStore without per-product special cases.

## Diagnostics

When a SigV4 request fails validation, the API server log dumps the inputs the validator used:

```
SigV4 MISMATCH: computed=… provided=…
SigV4 canonicalRequest:
   PUT
   /lake/year%3D2026/month%3D05/file.parquet
   …
SigV4 stringToSign:
   AWS4-HMAC-SHA256
   20260507T000000Z
   …
SigV4 signedHeaders=host;x-amz-content-sha256;x-amz-date, method=PUT, uri=…, queryString=…
SigV4 payloadHash=UNSIGNED-PAYLOAD
SigV4 headers passed: {host=…, x-amz-date=…, …}
```

Reproduce the canonical request on the SDK side and diff line-by-line — the first difference is the cause. Common culprits in the wild are clock skew (the SDK's `x-amz-date` differs from the server's by more than five minutes), path re-encoding (the SDK or a proxy in front of ShannonStore re-encoded a partition path), and missing signed headers (a request-id-injecting proxy added a header the SDK didn't include in `SignedHeaders`).

## See also

- [Identity & Access Management](iam.md) — user / group / policy / access-key entity model and the policy evaluator semantics.
- [S3-Compatible API](s3-compatible-api.md) — full dispatch table of routes that consume these authorization decisions.
- [Admin Password Recovery](admin-password-recovery.md) — out-of-band channel when the default-credential gate locks you out.
- [Nginx Reverse Proxy](../operations/nginx-proxy.md) — how to expose the data and admin ports without bleeding admin into the public internet.
