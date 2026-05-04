# Metadata Management

ShannonStore's object metadata (which chunk lives on which disk, sizes, ETags, versions) is distributed and replicated across the API tier so any node can serve any request and a single node failure never loses access to your objects.

## How keys are placed

Each `(bucket, key)` is owned by a fixed number of API nodes (default **2**). When a node joins or leaves the cluster, only the small fraction of keys whose ownership changed is moved — there is no full rebalance window or migration tool to run.

## Storage on each node

Metadata on each API node is split into multiple internal stores so that compaction and disk IO don't pile up on a single store as the cluster grows. These stores can be **striped across multiple physical disks** to scale IO further — useful for very large deployments.

## Replication & freshness

The default replication mode is **PULL**: writes commit on one node and are acknowledged immediately, while replicas catch up asynchronously. This gives the lowest write latency. Optional **synchronous** mode is available when stronger freshness is required.

## Membership changes

Adding or removing an API node automatically reshuffles only the affected keys. Joining nodes pull what they're now responsible for; leaving nodes' keys are immediately picked up by the surviving owners. An optional periodic sweep can clean up local copies of keys a node no longer owns (Cassandra-style cleanup) — disabled by default, opt-in from the admin UI.

## Reads

Reads are served locally if the node holds the key. Otherwise the node fetches it from the responsible peer transparently — clients never need to know which node holds what.
