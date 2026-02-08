# Architecture

ShannonStore is a high-performance, S3-compatible object storage system designed for high availability
and elastic scalability in modern data environments.

## ShannonStore Architecture

<img width="600" src="../../images/architecture/shannonstore-architecture.png" align="center"/>


Control & Logic Layer (API Servers) is the "Brain" of ShannonStore. It interprets the S3 protocol and orchestrates data movement.

* S3 Protocol Handling: Processes RESTful requests and generates S3-compliant XML/JSON responses.
* Metadata Management (RocksDB): Manages object locations, sizes, and permissions. Metadata is logically divided into 64+ Partitions and distributed across the API cluster.
* Security (KMS & IAM): Enforces granular Role-Based Access Control (RBAC) and ensures all data is encrypted at rest(SSE-S3, SSE-KMS support) and in transit using a distributed Key Management Service.
* Request Coordination:
    * Writes: Encodes incoming data into Erasure Coding (EC) shards (e.g., 2+1 or 4+2) and transmits them in parallel to Data Nodes.
    * Reads: Retrieves shards from multiple Data Nodes and reconstructs the original data via EC decoding.


Storage Layer (Data Nodes) is the "Warehouse" of the cluster, responsible for persisting the actual data fragments (chunks).

* Chunk Store: Physically stores binary fragments received from API servers on local disks.
* High-Speed I/O: Utilizes NIO (Non-blocking I/O) communication to handle high-throughput, large-scale data transfers efficiently.
* Agnostic Storage: Data nodes are "metadata-blind." They focus solely on storing and retrieving bytes associated with a Chunk ID, which maximizes the system’s horizontal scalability.


Coordination Layer (ZooKeeper) is the "Nervous System" that maintains cluster consensus and global state.

* Service Discovery: Tracks the real-time availability of all API and Data nodes.
* Leader Election: Elects a Cluster Controller (Leader API node) responsible for partition assignments and centralized action history.
* Sticky Partitioning: Persists the Partition Assignment Map. This ensures that even after a restart, partitions remain mapped to the same nodes, preventing unnecessary data migration.


ShannonStore has the following architectural advantages.

* Independent Scalability: Storage capacity (Data Nodes) and processing power (API Servers) can be scaled independently.
* Fault Tolerance: The system survives Data Node failures via Erasure Coding and API Node failures via ZooKeeper-based failover.
* Recoverable State: All critical states (IAM, KMS, History) are backed up to the Data Nodes, making API servers functionally stateless and easily replaceable.




