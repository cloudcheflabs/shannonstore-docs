# Distributed Architecture

ShannonStore follows a disaggregated architecture with clear separation between control plane and data plane.

## API Nodes (Control Plane)

- Handle S3 protocol requests, manage object metadata, and coordinate cluster operations (leader election, partition assignment, backup/restore).
- Perform erasure coding, encryption, and compression before distributing data to Data Nodes.
- Serve the Admin UI and Admin API on a dedicated port.

## Data Nodes (Data Plane)

- Store raw data chunks (erasure-coded shards) on local disks.
- Serve chunk read/write requests from API Nodes and report disk health to the cluster.
- Support multi-disk configurations for scalable storage capacity.

## Two-Port Architecture

- **S3 Port** (default 8080): High-throughput object storage operations.
- **Admin Port** (default 8888): Admin UI, REST API, and cluster management. Separated to avoid contention with the data path.
