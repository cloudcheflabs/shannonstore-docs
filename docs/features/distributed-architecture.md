# Distributed Architecture

ShannonStore follows a disaggregated architecture with clear separation between control plane and data plane.

## API Nodes (Control Plane)

- Handle S3 protocol requests, manage object metadata, and coordinate cluster operations (leader election, IAM/KMS state).
- Perform erasure coding, encryption, and compression before distributing data to Data Nodes.
- Use **HRW (Rendezvous Hashing) with a 65,536 vbucket layer** to assign each (bucket,key) to its top-R owner nodes for metadata replication, and to assign each chunk to its top-(D+P) data-node disks.
- Serve the Admin UI and Admin API on a dedicated port.

## Data Nodes (Data Plane)

- Store erasure-coded shards as plain files on local disks.
- Serve chunk read/write requests from API Nodes and report disk health to the cluster.
- Support multi-disk configurations for scalable storage capacity.

## Cluster coordination

- ZooKeeper holds the live membership (`/shannonstore-{api,data}`) and three readiness signals:
  - `/s3/leader-ready` — the leader has finished its own KMS/IAM init and peers may pull state from it.
  - `/s3/cluster-ready` — every API + data node has registered ready=true. S3 and admin paths reject requests with HTTP 503 until this flips true (a peer may still be initialising before this).
  - `/s3/coordinator-leader-id` — sticky leader hint so the same node tends to win re-elections after restart.
- Followers and data nodes block on `/s3/leader-ready` before pulling KMS/IAM from the leader.
- IAM mutations (login/password, IAM keys, bucket creation, STS) are forwarded to the leader so its RocksDB is the single source of truth; followers never push their own IAM snapshot.

## Two-Port Architecture

- **S3 Port** (default 8080): High-throughput object storage operations.
- **Admin Port** (default 8888): Admin UI, REST API, and cluster management. Separated to avoid contention with the data path.
