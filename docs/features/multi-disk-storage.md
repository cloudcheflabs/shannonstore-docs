# Multi-Disk Storage

ShannonStore Data Nodes support multiple local disks for scalable storage capacity.

- Chunks are evenly distributed across disks using hash-based placement for balanced utilization.
- Read and write operations include fallback mechanisms to handle disk topology changes gracefully.
- Disk health (capacity, usage, availability) is continuously tracked and reported to the cluster for placement decisions.
