# Distributed Architecture

ShannonStore separates the request/control plane from the data plane so the two scale independently.

## API Nodes (Control Plane)

- Terminate the S3 protocol and serve the Admin UI / Admin API on a dedicated port.
- Hold and replicate object metadata across the API tier; adding or removing an API node moves only the small fraction of keys whose ownership changed.
- Erasure-code, encrypt, and compress objects before distributing shards to data nodes; reconstruct them on read.

## Data Nodes (Data Plane)

- Store erasure-coded shards as plain files on local disks.
- Serve chunk read/write requests from API Nodes and report disk health back to the cluster.
- Support multi-disk configurations for scalable storage capacity.

## Cluster coordination

- A cluster opens to client traffic only when every API node is ready **and** at least one data node has registered with every registered data node ready. Until then, requests are rejected. There is no timeout — a cluster started without any data nodes simply stays closed.
- The previous leader tends to win re-election after a restart (sticky leadership), so cluster state moves as little as possible.
- IAM and KMS changes always run on the leader. Admin requests that land on a follower are automatically routed to the leader so a single source of truth is preserved.
- Default `admin/admin` credentials are created only on the very first cluster bootstrap. After that, the cluster will never recreate defaults — protecting against any local issue silently overwriting your real IAM.

## Two-Port Architecture

- **S3 Port** (default 8080): high-throughput object storage operations.
- **Admin Port** (default 8888): Admin UI, REST API, and cluster management. Separated from the S3 port so admin traffic never contends with the data path.
