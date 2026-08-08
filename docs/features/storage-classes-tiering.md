# Storage Classes & Tiering

ShannonStore exposes an S3-compatible **storage class registry** that doubles as the destination map for **lifecycle Transition** rules. Cold data is moved out of the hot cluster into a second S3 endpoint (another ShannonStore, MinIO, or AWS S3) on a schedule, and `GET` on a transitioned object responds with `403 InvalidObjectState` until the client issues `RestoreObject`.

## The registry

A storage class is a cluster-global record with an EC profile and an optional tier destination:

```json
{
  "name": "COLD",
  "description": "Cold tier for infrequently accessed data",
  "ecDataShards": 2,
  "ecParityShards": 1,
  "tierDestination": {
    "endpoint": "https://cold.example.com",
    "region": "us-east-1",
    "bucket": "lakehouse-cold",
    "accessKey": "AKIA...",
    "secretKey": "...",
    "kmsKeyId": "optional-informational-key-id"
  }
}
```

`description` and `tierDestination.kmsKeyId` are both optional fields on the `StorageClass` model.

### Admin endpoints

| Path | Method | Effect |
|---|---|---|
| `/admin/storage-classes` | GET | List all classes |
| `/admin/storage-classes/{name}` | GET | Single class |
| `/admin/storage-classes/{name}` | PUT | Create or replace |
| `/admin/storage-classes/{name}` | DELETE | Remove (rejected if any bucket points at it) |
| `/admin/buckets/{bucket}/storage-class` | GET | Bucket's assignment |
| `/admin/buckets/{bucket}/storage-class` | PUT | Assign a class to the bucket |
| `/admin/buckets/{bucket}/storage-class` | DELETE | Clear the assignment |

All mutations are **leader-only** — a write to a follower is proxied to the leader, and the leader broadcasts the updated registry to every follower synchronously, so a follower-routed GET right after the PUT already sees the new class.

The admin UI **Storage Classes** page covers create / edit / delete + per-bucket assignment. A class that is in use cannot be deleted.

## Per-bucket EC override

Setting a bucket's storage class also changes the **effective EC profile** for PUTs on that bucket. The profile (`ecDataShards`/`ecParityShards` of the assigned class) is stamped onto each new object's metadata, so reads stay correct even after a later cluster default change.

## Lifecycle Transition

Once a class is registered with a `tierDestination`, an S3 lifecycle Transition rule moves objects there. Either wire path works — both go through the same leader-validated path:

```xml
<LifecycleConfiguration>
  <Rule>
    <ID>tier-cold</ID>
    <Status>Enabled</Status>
    <Filter><Prefix></Prefix></Filter>
    <Transition>
      <Days>1</Days>
      <StorageClass>COLD</StorageClass>
    </Transition>
  </Rule>
</LifecycleConfiguration>
```

| Wire | Path |
|---|---|
| S3 | `PUT /<bucket>?lifecycle` (XML body) |
| Admin alias | `PUT /admin/buckets/<bucket>/lifecycle` (XML body) |
| Admin (bucket-config) | `PUT /admin/browser/bucket-config/<bucket>/lifecycle` |

The leader validates the rule's `StorageClass` against the registry **at PUT time** — referencing an unregistered class throws immediately rather than silently no-oping on the next scan cycle — but the HTTP status differs by wire path: only the admin alias (`PUT /admin/buckets/<bucket>/lifecycle`) catches the error and returns `400 Bad Request`. The S3-native path and the admin bucket-config path both surface it as `500 Internal Server Error` instead. (ShannonStore extension: `<Seconds>N</Seconds>` is accepted alongside the AWS-standard `<Days>` / `<Date>` for sub-day testing.)

### Lifecycle scanner

`LifecycleExpiryService` is a leader-only worker that periodically scans buckets, matches each object against the policy, and:

- For `Expiration`: drops the object outright.
- For `Transition`: streams the decrypted bytes to the destination, marks `meta.storageClass / tierLocation / tieredAt`, and removes the local EC chunks (metadata remains, so the object is still listable).
- For `AbortIncompleteMultipartUpload`: drops parts older than the threshold.

Status + manual trigger from the admin UI **Maintenance → Lifecycle** page, or:

```bash
curl -sS -H "Authorization: Bearer $TOK" http://api:8888/admin/maintenance/lifecycle/status
curl -sS -X POST -H "Authorization: Bearer $TOK" http://api:8888/admin/maintenance/lifecycle/trigger
```

## GET on a tiered object — and RestoreObject

`GET /<bucket>/<key>` for a tiered object returns:

```http
HTTP/1.1 403 Forbidden
Content-Type: application/xml

<Error><Code>InvalidObjectState</Code>...</Error>
```

`HEAD /<bucket>/<key>` is fine — it returns `x-amz-storage-class: COLD` (and `x-amz-restore` if a restore is in flight).

To recall hot bytes, send `POST /<bucket>/<key>?restore` with `<RestoreRequest><Days>7</Days></RestoreRequest>` — the leader's `RestoreObjectService` pulls the object back from the tier destination, places it across the hot cluster, and stamps `restoreExpiresAt = now + 7d`. While the worker runs, `restoreStatus = IN_PROGRESS`; afterwards `COMPLETED` and `GET` succeeds again. Once `restoreExpiresAt` elapses, the lifecycle scanner re-tiers the object.

## Async cleanup on DELETE

When a tiered object is deleted, the local metadata is dropped immediately (so the DELETE returns fast) and the destination object is queued for cleanup on the leader's `TieredCleanupService`. The queue has exponential back-off and a dead-letter list — failures don't block client DELETEs.

## End-to-end example

`tests/test-tier-e2e-two-clusters.sh` ships a two-cluster docker-compose scenario that walks the entire flow:

1. Register `COLD` class with a `tierDestination` pointing at site B.
2. Apply a lifecycle Transition rule with `<Seconds>2</Seconds>`.
3. After 4 s, trigger the scanner — the object moves to site B.
4. `HEAD` shows `x-amz-storage-class: COLD`; `GET` returns 403.
5. `POST ?restore` pulls it back; `GET` succeeds.
6. `DELETE` triggers async cleanup on site B.

```bash
bash tests/test-tier-e2e-two-clusters.sh
```
