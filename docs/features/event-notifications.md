# Object Event Notifications (Webhook & Kafka)

ShannonStore can push a message to an external system every time an object changes —
the same capability MinIO bucket notifications and AWS S3 Event Notifications provide.
When a client `PUT`s, copies, completes a multipart upload of, or deletes an object,
ShannonStore emits a small JSON event and delivers it to the sinks you configured:
an **HTTP webhook** and/or a **Kafka topic**.

```text
   client ──PUT s3://logs/app/2026.json──▶  ShannonStore (S3 :8080)
                                                  │  write commits
                                                  ▼
                                        ObjectEventDispatcher (per node)
                                          match rules ─┬─▶ Webhook  POST https://hook/…
                                                       └─▶ Kafka    produce → topic "s3-events"
```

Typical uses: kick off a downstream pipeline when a file lands, index new objects in a
search system, drive a Kafka→Flink/Spark stream, invalidate a cache on delete, or feed
an audit log.

## Model

The design mirrors MinIO's two-level model:

- **Targets** — the delivery sinks. A target is either a `webhook` (an HTTP endpoint)
  or a `kafka` producer (bootstrap servers + topic). Targets are **cluster-global** and
  managed from the Admin UI.
- **Rules** — bind a set of events to a target, with an optional bucket / prefix /
  suffix filter. A rule says *"for `s3:ObjectCreated:*` on bucket `logs` under prefix
  `app/` ending in `.json`, deliver to target `kf1`."*

Both live in one cluster-global configuration that every node reads, so **whichever
node serves a write emits with identical routing**. It is stored in the replicated,
RocksDB-backed bucket configuration (see
[Bucket Configuration → Cluster consistency](bucket-configuration.md#cluster-consistency-leader-routed-writes)),
so it **survives a full cluster restart**.

Notifications are **disabled by default** — nothing is emitted until an operator turns
the master switch on, exactly like the other operator features (bitrot scrubber,
lifecycle expiry).

## Event types

| Event name | Fired when |
| --- | --- |
| `s3:ObjectCreated:Put` | a single `PutObject` commits |
| `s3:ObjectCreated:CompleteMultipartUpload` | a multipart upload is completed |
| `s3:ObjectCreated:Copy` | a `CopyObject` writes the destination |
| `s3:ObjectRemoved:Delete` | an object (or version) is deleted |
| `s3:TestEvent` | the Admin UI **Test** button sends a synthetic event |

A rule's event list accepts exact names (`s3:ObjectCreated:Put`), a family wildcard
(`s3:ObjectCreated:*`), or `*` for everything. An empty list also means "all events".

## Event payload (AWS-S3-compatible)

The body is intentionally shaped like the AWS S3 notification record, so existing
S3-event consumers (Lambda-style handlers, MinIO tooling) work unchanged:

```json
{
  "Records": [
    {
      "eventVersion": "2.1",
      "eventSource": "shannonstore:s3",
      "awsRegion": "us-east-1",
      "eventTime": "2026-07-09T12:43:03.749Z",
      "eventName": "s3:ObjectCreated:Put",
      "userIdentity": { "principalId": "00BD104E21E447F0" },
      "requestParameters": { "sourceIPAddress": "10.0.0.5" },
      "s3": {
        "s3SchemaVersion": "1.0",
        "configurationId": "shannonstore-notification",
        "bucket": {
          "name": "logs",
          "ownerIdentity": { "principalId": "00BD104E21E447F0" },
          "arn": "arn:aws:s3:::logs"
        },
        "object": {
          "key": "app/2026.json",
          "size": 1024,
          "eTag": "46d25cd5ffe2eecb2896fa9a51053f93",
          "versionId": "…",
          "sequencer": "19F469236DE000BF"
        }
      }
    }
  ]
}
```

- `eTag` is ShannonStore's MD5 ETag (single-PUT = MD5 of the plaintext; multipart =
  MD5-of-part-MD5s + `-N`). Absent on delete events.
- `userIdentity.principalId` is the requesting access key.
- For Kafka, the record key is `bucket/key`, so all events for one object land on the
  same partition and stay ordered.

## Delivery semantics — best-effort, non-blocking

The dispatcher never sits on the S3 request path:

- `emit()` only **offers the event to a bounded in-memory queue** and returns; the
  client response is not delayed by delivery.
- A fixed pool of worker threads drains the queue, matches rules, and delivers with
  **bounded retry + exponential backoff**.
- If the queue is **full** (a target is down and events pile up), new events are
  **dropped** (counted, throttled-warn) rather than blocking writes — an at-most-once
  fallback under overload.
- Otherwise delivery is **at-least-once, best-effort**: duplicates are possible on
  retry, and ordering is only guaranteed per object per Kafka partition.

This matches the guarantees of AWS S3 / MinIO notifications and keeps a slow or dead
consumer from ever affecting client latency.

!!! note "Per-node counters"
    Each node runs its own dispatcher and counters. Behind a load balancer the events
    for a workload are spread across nodes, so the Admin UI **Delivery Stats** panel
    and the Prometheus counters are **per node** — sum across nodes for a cluster total.

## Configure from the Admin UI

Open **Event Notifications** in the Admin console:

1. **Add a target.** Choose *Webhook* (enter the URL, optional `Authorization`
   header) or *Kafka* (bootstrap servers + topic).
2. **Test it.** The **Test** button delivers a synthetic `s3:TestEvent` synchronously
   and reports success or the exact error — verify connectivity before relying on it.
3. **Add a rule.** Pick the target, a bucket (or `*` for all), optional prefix/suffix,
   and the event types.
4. **Turn on** the master switch and **Save**. Config propagates cluster-wide within a
   few seconds.

!!! warning "Avoid feedback loops"
    If another product stores its data **in ShannonStore** (e.g. a Kafka-compatible
    broker whose segments live in a ShannonStore bucket), do **not** route that
    bucket's create events into that same broker — the broker's own segment writes
    would generate events that it re-ingests, and so on. Scope the rule to the specific
    application bucket instead of `*`.

## Configure via the admin REST API

The whole configuration is one JSON document (`GET`/`PUT`):

```bash
# Read the current config
curl http://node:8888/admin/notifications/config \
  -H "Authorization: Bearer <admin-jwt>"

# Replace it: one webhook + one kafka target, two rules, enabled
curl -X PUT http://node:8888/admin/notifications/config \
  -H "Authorization: Bearer <admin-jwt>" -H "Content-Type: application/json" -d '{
    "enabled": true,
    "targets": [
      { "id": "wh1", "type": "webhook", "enabled": true,
        "webhookUrl": "https://pipeline.example.com/s3-hook",
        "webhookHeaders": { "Authorization": "Bearer <secret>" } },
      { "id": "kf1", "type": "kafka", "enabled": true,
        "kafkaBootstrapServers": "broker-1:9092,broker-2:9092",
        "kafkaTopic": "s3-events" }
    ],
    "rules": [
      { "id": "r1", "enabled": true, "targetId": "wh1",
        "bucket": "uploads", "prefix": "incoming/", "suffix": ".csv",
        "events": ["s3:ObjectCreated:*"] },
      { "id": "r2", "enabled": true, "targetId": "kf1",
        "bucket": "logs", "events": ["s3:ObjectCreated:*", "s3:ObjectRemoved:Delete"] }
    ]
  }'

# Fire a synthetic test event at one target
curl -X POST http://node:8888/admin/notifications/test \
  -H "Authorization: Bearer <admin-jwt>" -H "Content-Type: application/json" \
  -d '{ "targetId": "kf1" }'

# Per-node delivery stats
curl http://node:8888/admin/notifications/stats -H "Authorization: Bearer <admin-jwt>"
```

Writes are leader-owned: a `PUT` on any node is forwarded to the leader, applied, and
replicated to every node's bucket config.

### Kafka target options

`kafkaProducerProps` on a Kafka target passes arbitrary producer properties through to
the underlying client — use it for security:

```json
{ "id": "kf-secure", "type": "kafka", "enabled": true,
  "kafkaBootstrapServers": "broker:9093", "kafkaTopic": "s3-events",
  "kafkaProducerProps": {
    "security.protocol": "SASL_SSL",
    "sasl.mechanism": "PLAIN",
    "sasl.jaas.config": "org.apache.kafka.common.security.plain.PlainLoginModule required username=\"u\" password=\"p\";"
  } }
```

The producer publishes with `acks=all` and fails fast (bounded `max.block.ms` /
delivery timeout) so a broker outage surfaces as a counted failure the dispatcher can
retry, rather than blocking a worker.

## Tuning (shannonstore.properties)

The per-node dispatcher is tuned in `shannonstore.properties` §5b (the targets/rules
themselves live in the replicated bucket config, not here):

```properties
# Bounded dispatch queue per node; when full, events are dropped (counted) not blocked.
shannonstore.api.notification.queue.size=10000
# Worker threads draining the queue and delivering to targets.
shannonstore.api.notification.worker.threads=4
# Max delivery retries per event before it is counted as failed (0 = no retry).
shannonstore.api.notification.max.retries=3
# Base backoff (ms) between retries — exponential, capped at 5s.
shannonstore.api.notification.retry.backoff.ms=200
# How often (s) each node re-reads the replicated config so changes take effect.
shannonstore.api.notification.config.poll.seconds=5
```

## Metrics

The dispatcher exposes Prometheus counters + a queue-depth gauge on each node's
`/metrics` (admin port). See
[Monitoring & Metrics → Object event notifications](monitoring.md#object-event-notifications)
for the full list and a suggested alert on drops.

## Related

- [Bucket Configuration](bucket-configuration.md) — the durable, leader-routed store
  the notification config lives in.
- [Monitoring & Metrics](monitoring.md) — dispatcher metrics and alerts.
