# Maintenance Mode

ShannonStore supports a global maintenance mode for safe cluster-wide operations.

## Activation

- Toggled via Admin API: POST /admin/maintenance/mode?enabled=true|false.
- Broadcasts TYPE_MAINTENANCE_MODE_REQ to all API Nodes in the cluster.

## Behavior

- When active, all S3 API requests return 503 Service Unavailable with Retry-After: 30 header.
- Admin API operations continue to function normally.
- Part buffer flush can be triggered to ensure all buffered data is persisted before maintenance.

## Use Cases

- Partition rebalancing (prevents data inconsistency during reassignment).
- Cluster-wide backup operations.
- Part of the Smart Rebalance workflow (phase 1: lock → phase 5: unlock).
