# Maintenance Mode

ShannonStore supports a global maintenance mode for safe cluster-wide operations.

- When activated, all S3 API requests are temporarily rejected while admin operations remain available.
- Used during partition rebalancing, cluster-wide backups, and other operations that require consistent cluster state.
