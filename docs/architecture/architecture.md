# Architecture

ShannonStore is a high-performance, S3-compatible object storage system designed for high availability and elastic scalability in modern data environments. The cluster is composed of three logical tiers — a control & logic tier, a storage tier, and a coordination tier — each scalable independently.

## ShannonStore Architecture

<img width="600" src="../../images/architecture/shannonstore-architecture.png" align="center"/>

### Control & Logic Layer (API Servers)

API Servers are the "brain" of ShannonStore. They terminate the S3 protocol, enforce security, and orchestrate data movement across the storage tier.

- **S3 protocol handling** — Standard S3 over HTTPS: SigV4 authentication, multipart upload, list-objects, and STS AssumeRole.
- **Metadata management** — Object location and metadata is distributed across the API tier and replicated for durability. Adding or removing an API node moves only the small fraction of keys whose ownership changed — no cluster-wide rebalance window.
- **Security** — Role-based access control through users, groups, and policies. Data is encrypted at rest (SSE-S3, SSE-KMS) and in transit, with key material wrapped by a cluster-wide master key.
- **Erasure coding** — Each object is split into data + parity shards on write and reconstructed from any healthy subset on read. Disk loss is transparent to the client.

### Storage Layer (Data Nodes)

Data Nodes are the "warehouse" of the cluster. They persist binary fragments to local disks and serve them to API Servers on demand.

- Stored on plain files across multiple disks for high throughput and predictable behaviour.
- Concurrent shard transfers via an asynchronous network layer.
- Data Nodes know nothing about object keys, ACLs, or encryption metadata — they store and retrieve bytes by chunk identifier. This separation lets the storage tier scale independently of the request tier.
- A background scrubber checks data integrity over time and repairs silent corruption automatically.

### Coordination Layer (ZooKeeper)

ZooKeeper provides cluster membership, leader election, and the readiness signal that gates client traffic.

- **Service discovery** — every node registers itself and its readiness state.
- **Sticky leader election** — exactly one API node is elected leader; on restart, the previous leader tends to win re-election so the cluster minimises state movement.
- **Cluster opening guarantees**:
    - Traffic is accepted only after every API node is ready **and** at least one data node has registered with every registered data node ready. Until then, requests are rejected. There is no timeout — a cluster started without any data nodes simply stays closed.
    - Default `admin/admin` credentials are created only on the very first bootstrap, never again. A leader whose local state looks empty for any local reason refuses to recreate defaults — your real IAM cannot be silently overwritten.

### Architectural Properties

- **Independent scalability** — Storage capacity (Data Nodes) and request + metadata capacity (API Servers) scale independently. Membership changes move only the affected slice of data.
- **Fault tolerance** — Data Node and disk failures are masked by erasure-coded parity. API Server failures trigger automatic re-election; surviving replicas cover the lost node's metadata until it returns.
- **Single source of truth** — IAM and KMS state always live on the leader. Followers and Data Nodes synchronise from the leader on every restart and on every change at runtime.
- **Pluggable disaster recovery** — In-cluster system-state backup has been removed in favour of an external destination (e.g., a different S3 bucket), aligning disaster recovery with standard offsite-backup practice.
