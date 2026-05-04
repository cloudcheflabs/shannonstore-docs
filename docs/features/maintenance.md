# Maintenance Mode

ShannonStore supports a global maintenance mode for safe cluster-wide operations.

- When activated, S3 client writes are temporarily paused while admin tooling stays available, giving operators a stable window to inspect or service the cluster.
- Used to drain in-flight work before disruptive operations such as scheduled disk replacement or evacuating a node.
- Adding or removing API and Data Nodes does **not** require maintenance mode — the cluster rebalances ownership automatically. Maintenance mode is a defensive lever, not part of routine scaling.
