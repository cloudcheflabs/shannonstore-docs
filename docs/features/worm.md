# Object Lock (WORM)

ShannonStore implements S3 Object Lock — the standard write-once-read-many guarantee that makes a bucket suitable for compliance, audit, and regulatory workloads. Once an object is locked, **no caller can delete or overwrite it** until the retention window expires, regardless of IAM privilege. The implementation matches the AWS S3 Object Lock spec on the wire so SDKs configured for AWS work unchanged.

```text
   PUT /bucket
     x-amz-bucket-object-lock-enabled: true
                │
                ▼
   ┌──────────────────────────────────────────┐
   │ Bucket has Object Lock                    │
   │   ├── versioning forced ON                │
   │   ├── optional default retention rule     │
   │   │     (GOVERNANCE | COMPLIANCE + days)  │
   │   └── PUT object can attach:              │
   │       ─ x-amz-object-lock-mode            │
   │       ─ x-amz-object-lock-retain-until-date│
   │       ─ x-amz-object-lock-legal-hold      │
   └────────────────┬─────────────────────────┘
                    │
                    ▼
   ┌──────────────────────────────────────────┐
   │ Each object carries:                      │
   │   retentionMode    ("GOVERNANCE"|"COMPLIANCE"|null)
   │   retainUntilDate  (epoch ms; 0 = none)   │
   │   legalHoldOn      (boolean)              │
   └────────────────┬─────────────────────────┘
                    │
                    ▼  DELETE attempt
   ┌──────────────────────────────────────────┐
   │ Enforcement:                              │
   │   Legal Hold ON    → 403 (never bypass)   │
   │   COMPLIANCE active→ 403 (never bypass)   │
   │   GOVERNANCE active→ 403 unless           │
   │      x-amz-bypass-governance-retention=true
   │      AND s3:BypassGovernanceRetention     │
   │      IAM permission granted               │
   │   Retention expired + no hold → DELETE OK │
   └──────────────────────────────────────────┘
```

The enforcement is **at the SCAN-handler level inside the API node** — before any storage call. There is no client-side rule the operator has to remember; if the cluster denies a deletion, the client gets a clean `403 AccessDenied` with a message naming the protection class and (for retention) the expiry timestamp.

## When to use

| Use it for | Don't use it for |
| --- | --- |
| SEC 17a-4(f), HIPAA, GDPR audit trails — regulators care about *who can delete what* | Day-to-day backup retention (lifecycle expiry handles this cheaper) |
| Litigation hold — pin specific evidence while a case is open | "Delete protection" for important objects — IAM Deny on `s3:DeleteObject` is the right tool |
| Tamper-proof log archives that any compromised credential must not be able to wipe | Per-team "keep these around" requests that get adjusted often |
| Cross-region replication targets that must outlive the source | Mixed-policy buckets — once Object Lock is on, it cannot be turned off |

The single design choice that matters: **Object Lock is enabled only at bucket creation time** and can never be turned off. A bucket that *might* need locked objects later still has to be created with `x-amz-bucket-object-lock-enabled: true`. Default retention can be added or changed afterwards, but the bucket-level enable flag is one-way.

## Bucket-level configuration

### Enable Object Lock at creation

```bash
curl -X PUT "http://localhost:8080/audit-logs" \
  -H "x-amz-bucket-object-lock-enabled: true" \
  --user "$AK:$SK" --aws-sigv4 "aws:amz:us-east-1:s3"
```

What the cluster does:

1. Creates the bucket.
2. **Forces versioning ON** for the bucket — Object Lock requires versioning per S3 spec, and the cluster enforces this implicitly rather than asking the operator to remember a second flag.
3. Persists an `ObjectLockConfig{ enabled: true, defaultRetention: null }` on the bucket and broadcasts the change to peer API nodes.

A subsequent `PUT /<bucket>?versioning` with `<Status>Suspended</Status>` is *rejected* on a lock-enabled bucket — suspending versioning would undermine the lock guarantee.

### Set or update the default retention rule

A default retention rule auto-applies to every new object placed in the bucket. Configure via `PUT /<bucket>?object-lock`:

```bash
curl -X PUT "http://localhost:8080/audit-logs?object-lock" \
  --data '<ObjectLockConfiguration>
            <ObjectLockEnabled>Enabled</ObjectLockEnabled>
            <Rule>
              <DefaultRetention>
                <Mode>GOVERNANCE</Mode>
                <Days>7</Days>
              </DefaultRetention>
            </Rule>
          </ObjectLockConfiguration>' \
  --user "$AK:$SK" --aws-sigv4 "aws:amz:us-east-1:s3"
```

`Days` and `Years` are mutually exclusive — pick one. `Mode` is `GOVERNANCE` or `COMPLIANCE` (see [Retention modes](#retention-modes) below).

Operator can also leave the default rule off and assign retention per-object at PUT time. Both paths produce identical per-object state.

A `GET /<bucket>?object-lock` returns the current config or `404 ObjectLockConfigurationNotFoundError` when the bucket has Object Lock disabled.

### What if I try `?object-lock` on a non-lock bucket?

`409 InvalidBucketState` — the lock flag is creation-time only, and the cluster rejects rather than silently enabling something the AWS spec doesn't allow.

## Per-object retention and legal hold

Every object in a lock-enabled bucket can carry its own retention rule, its own legal hold, or both. Set them at PUT time via headers, or modify after via subresource endpoints.

### Headers at PUT time

```bash
RET_UNTIL=$(date -u -v+30d "+%Y-%m-%dT%H:%M:%SZ" 2>/dev/null \
            || date -u -d "+30 day" "+%Y-%m-%dT%H:%M:%SZ")

curl -X PUT "http://localhost:8080/audit-logs/2026-05-29.log" \
  -H "x-amz-object-lock-mode: GOVERNANCE" \
  -H "x-amz-object-lock-retain-until-date: $RET_UNTIL" \
  -H "x-amz-object-lock-legal-hold: OFF" \
  --upload-file ./local-log \
  --user "$AK:$SK" --aws-sigv4 "aws:amz:us-east-1:s3"
```

| Header | Values | Default when omitted |
| --- | --- | --- |
| `x-amz-object-lock-mode` | `GOVERNANCE` / `COMPLIANCE` | the bucket's default rule if set, else none |
| `x-amz-object-lock-retain-until-date` | ISO 8601 UTC | derived from the bucket's default rule (now + days/years) |
| `x-amz-object-lock-legal-hold` | `ON` / `OFF` | `OFF` |

If both the headers and a bucket default rule are present, **the headers win** — the per-object override is total, not merged.

### Modify retention after the fact

```bash
# Extend the retention by another 30 days — strengthening, always allowed.
curl -X PUT "http://localhost:8080/audit-logs/2026-05-29.log?retention" \
  --data "<Retention>
            <Mode>GOVERNANCE</Mode>
            <RetainUntilDate>${LATER}</RetainUntilDate>
          </Retention>" \
  --user "$AK:$SK" --aws-sigv4 "aws:amz:us-east-1:s3"
```

`GET /<key>?retention` returns the current rule. `404 NoSuchObjectLockConfiguration` when no rule is set.

### Toggle legal hold

```bash
# Pin an object — survives retention expiry until cleared.
curl -X PUT "http://localhost:8080/audit-logs/2026-05-29.log?legal-hold" \
  --data '<LegalHold><Status>ON</Status></LegalHold>' \
  --user "$AK:$SK" --aws-sigv4 "aws:amz:us-east-1:s3"

# Release the hold.
curl -X PUT "http://localhost:8080/audit-logs/2026-05-29.log?legal-hold" \
  --data '<LegalHold><Status>OFF</Status></LegalHold>' \
  --user "$AK:$SK" --aws-sigv4 "aws:amz:us-east-1:s3"
```

Legal hold is orthogonal to retention. An object with retention expired but legal hold ON is still un-deletable; an object with no retention but legal hold ON is also un-deletable. The hold has no expiry — it stays until an authorized caller flips it to `OFF`.

## Retention modes

### GOVERNANCE

The everyday production mode. Once set, the retention applies to every caller — including the bucket owner — but a caller holding the dedicated **`s3:BypassGovernanceRetention`** IAM permission can request bypass by sending `x-amz-bypass-governance-retention: true` on the DELETE (or retention-loosening PUT).

Both conditions are required:

| Condition | Effect alone |
| --- | --- |
| Header only, no IAM permission | `403 AccessDenied` with message "s3:BypassGovernanceRetention permission required to override retention" |
| IAM permission only, no header | `403 AccessDenied` with message "resend with x-amz-bypass-governance-retention: true to override" |
| Both header + IAM permission | bypass succeeds; the bypass is logged at INFO with the userId, key, and original `retainUntilDate` |

The split protects against two distinct failure modes: a runaway script that knows the IAM grant exists can't accidentally bypass without the explicit header; an attacker with a stolen token can't bypass without first getting the bypass permission attached.

A representative IAM policy for a compliance-officer role:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "GovernanceBypassForCompliance",
      "Effect": "Allow",
      "Action": [
        "s3:DeleteObject",
        "s3:BypassGovernanceRetention"
      ],
      "Resource": [
        "arn:aws:s3:::audit-logs",
        "arn:aws:s3:::audit-logs/*"
      ]
    }
  ]
}
```

### COMPLIANCE

The stronger mode — **no caller can bypass**, including the cluster's root admin. The `x-amz-bypass-governance-retention` header is ignored (the bypass IAM permission has no effect). The only way a COMPLIANCE-mode object becomes deletable is by waiting until `retainUntilDate` passes.

COMPLIANCE also rejects *shortening* the retention window before it expires. A `PUT /<key>?retention` with an earlier `RetainUntilDate` than the current one returns `403 AccessDenied "Cannot loosen COMPLIANCE retention before expiry"`. Lengthening is always allowed.

Mode upgrades have the same asymmetry:

| Current mode | New mode | Allowed? |
| --- | --- | --- |
| none | GOVERNANCE | yes |
| none | COMPLIANCE | yes |
| GOVERNANCE | COMPLIANCE | yes (strengthening) |
| COMPLIANCE | GOVERNANCE | **no** — would weaken a compliance-grade lock; rejected with `403` |
| GOVERNANCE | GOVERNANCE w/ longer retain | yes |
| GOVERNANCE | GOVERNANCE w/ shorter retain | bypass required (header + IAM) |

Pick COMPLIANCE only when the workload demands strict tamper resistance. Once on a key, that key is genuinely write-once until the clock runs out — operationally this needs a robust automation around when retention expires and what happens to the object afterwards (lifecycle deletion vs. archival vs. manual review).

## Enforcement at DELETE

`DELETE /<bucket>/<key>` (and `DELETE` with `?versionId=…`) walks this decision tree:

1. Look up the object's metadata for the requested version.
2. Evaluate `isWormProtected(now)`:
    - legal hold ON → protected
    - retentionMode set and retainUntilDate > now → protected
    - otherwise → not protected
3. If not protected → normal delete.
4. If legal hold ON → `403 AccessDenied "Object is protected by a Legal Hold"`. Path ends.
5. If COMPLIANCE before expiry → `403 AccessDenied "Object is under a COMPLIANCE-mode retention until …"`. Path ends.
6. If GOVERNANCE before expiry:
    - no `x-amz-bypass-governance-retention: true` → `403 AccessDenied "… resend with x-amz-bypass-governance-retention: true to override"`
    - bypass header present but caller lacks `s3:BypassGovernanceRetention` → `403 AccessDenied "s3:BypassGovernanceRetention permission required …"`
    - both present → delete proceeds; INFO log records the bypass

`DELETE /<bucket>` (bucket itself) refuses while any object in the bucket is WORM-protected, returning `409 BucketNotEmpty` with the first protected key named. The check scans the in-memory index and short-circuits at the first protected hit, so a single protected object is enough to block bucket deletion regardless of how empty the rest of the bucket is.

## Versioning interaction

Object Lock requires versioning. The cluster enforces this implicitly when the bucket is created with `x-amz-bucket-object-lock-enabled: true`:

- `PUT /<bucket>?versioning` with `<Status>Suspended</Status>` on a lock-enabled bucket → `409 InvalidBucketState`. Suspending would let a subsequent overwrite silently bypass the lock on the *new* version while the old, locked version still exists in metadata but isn't the canonical answer.
- A `PUT` against a key that already has a locked version creates a *new* version (versioning is on) without disturbing the old one. The old version's retention applies; the new version's retention is whatever the headers or default rule set, or none.
- A `DELETE` without `versionId` writes a delete marker — this is allowed on a lock-enabled bucket because it doesn't remove any locked bytes; the underlying locked versions remain readable via `?versionId=…`.

The lifecycle / Object Lock interaction follows the AWS rule: a lifecycle expiration that fires while a version is locked is **silently skipped** (the lifecycle expiry scanner respects `isWormProtected(now)`). This guarantee composes — an operator can run lifecycle policies on a lock-enabled bucket without worrying that aggressive expiry rules will accidentally evict protected versions.

## IAM action verbs

Object Lock introduces seven distinct IAM verbs in the `s3:` namespace. Each maps to one wire path:

| Verb | Triggered by |
| --- | --- |
| `s3:PutBucketObjectLockConfiguration` | `PUT /<bucket>?object-lock`, `DELETE /<bucket>?object-lock` |
| `s3:GetBucketObjectLockConfiguration` | `GET /<bucket>?object-lock` |
| `s3:PutObjectRetention` | `PUT /<key>?retention`, `PUT /<key>` with retention headers |
| `s3:GetObjectRetention` | `GET /<key>?retention` |
| `s3:PutObjectLegalHold` | `PUT /<key>?legal-hold`, `PUT /<key>` with `x-amz-object-lock-legal-hold` |
| `s3:GetObjectLegalHold` | `GET /<key>?legal-hold` |
| `s3:BypassGovernanceRetention` | required *in addition to* `s3:DeleteObject` / `s3:PutObjectRetention` when bypassing GOVERNANCE |

Build policies that name these verbs explicitly. A blanket `s3:*` matches them by wildcard, but production IAM should grant the bypass verb separately so it can be audited and rotated independently from generic write access. See [IAM](iam.md) for the policy schema and evaluator semantics.

## Operational guidance

- **Plan the legal-hold workflow before going live**. The most common operational failure is a bucket that ends up with a thicket of legal holds nobody remembers to release. Track legal-hold issuance through a ticketing system and audit `shannonstore_legal_hold_set_total` quarterly.
- **GOVERNANCE for routine compliance, COMPLIANCE for true tamper-resistance**. Treat COMPLIANCE as a one-way door — pick it only when the workload genuinely demands that even the root admin be unable to delete.
- **Don't enable Object Lock on a bucket "just in case"**. The flag is one-way and force-enables versioning, which doubles the metadata footprint for most workloads. Buckets that aren't compliance targets cost more without giving anything back.
- **Audit `s3:BypassGovernanceRetention` grants**. The grant should sit on a small number of well-known roles (compliance officers, security incident responders) and never on application credentials. Cluster INFO logs name every bypass event with userId + key + original retain-until-date — retain those logs for the same window as the bucket's retention rule.
- **Watch for `BucketNotEmpty` when decommissioning**. A bucket can become un-deletable for years if a single locked object stays under COMPLIANCE retention; the operator must either wait, release legal holds, or migrate the data to a different lock-enabled bucket first.
- **Cross-region replication**. When the destination bucket also has Object Lock enabled with the same default rule, replicated objects arrive with their retention pre-attached. If the destination has *no* lock, the protection is silently dropped — verify both endpoints carry the same lock configuration before turning replication on.

## Troubleshooting

| Symptom | Likely cause + fix |
| --- | --- |
| `409 InvalidBucketState` on `PUT /<bucket>?object-lock` | The bucket was not created with `x-amz-bucket-object-lock-enabled: true`. Recreate the bucket with the header set — there is no retro-fit path. |
| `403 AccessDenied` on DELETE with bypass header set | Caller lacks `s3:BypassGovernanceRetention`. Add the verb to a small, audited role; do not put it on the same role that holds `s3:DeleteObject` for everyday writes. |
| `403 AccessDenied "Cannot loosen COMPLIANCE retention"` | COMPLIANCE is genuinely uncrackable. The only fix is to wait until `retainUntilDate` passes. |
| `409 BucketNotEmpty` on `DELETE /<bucket>` with apparently empty bucket | A version (not the current pointer) is still locked. `GET /<bucket>?versions` lists every version including delete markers; locate the locked version and either wait, bypass, or release its hold. |
| Lifecycle expiry counter rising but locked objects still present | Expected — the scanner skips WORM-protected objects. The `shannonstore_lifecycle_objects_skipped_worm_total` counter is the explicit signal that the lifecycle saw and skipped them. |
| Header `x-amz-bypass-governance-retention: true` ignored on COMPLIANCE object | By design — COMPLIANCE has no bypass. Check the object's `retentionMode` via `GET /<key>?retention` to confirm. |
| New version of a locked object is itself unlocked | Object Lock applies per-version. The new PUT writes a fresh version; its retention comes from the headers or the bucket default rule. Old versions stay locked. |

## See also

- [S3-Compatible API](s3-compatible-api.md) — the dispatch table this protection sits inside.
- [Bucket Configuration](bucket-configuration.md) — the lifecycle expiry scanner defers to these locks; a WORM-protected object is never expired.
- [IAM](iam.md) — policy schema, evaluator semantics, the `s3:BypassGovernanceRetention` verb in context.
- [Authentication & Authorization](auth-authz.md) — SigV4 / request authorization that runs before the lock check.
- [Backup & Restore](backup.md) — backup respects retention; restored objects come back with their locks intact.
- [Monitoring & Metrics](monitoring.md) — the WORM-related counters mentioned in the operational guidance.
