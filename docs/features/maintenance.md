# Maintenance Mode

ShannonStore supports a global maintenance mode for safe cluster-wide operations.

- When activated, all S3 API requests are temporarily rejected (HTTP 503 with `Retry-After`) while admin operations remain available.
- Used to flush in-flight buffers (`/admin/maintenance/flush`) before disruptive operations and to gate the cluster while administrators inspect or repair state.
- Membership changes (adding/removing API or data nodes) do **not** require maintenance mode — HRW + vbucket placement recomputes ownership automatically and the metadata pull cycle / bootstrap snapshot fills the new node in. Maintenance mode is a defensive lever, not a routine operation.
