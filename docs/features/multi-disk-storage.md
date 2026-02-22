# Multi-Disk Storage

## Hash-Based Chunk Distribution

- Data Nodes distribute chunks across multiple local directories using: index = (chunkId.hashCode() & 0x7FFFFFFF) % rootDirs.length.
- This ensures even load balancing across disks without centralized coordination.

## Fallback Mechanisms
- Read: Tries the expected hash-based location first; falls back to scanning all directories if not found (handles disk topology changes).
- Write: Uses hash-based selection by default; supports targeted disk writes via FLAG_DISK_AWARE for repair operations.
- Migration: migrateChunkToCorrectLocation() moves chunks to their correct hash-based location after disk count changes.

## Disk Health Tracking

- Data Nodes periodically report disk information to ZooKeeper via NodePayload.DiskInfo: path, total capacity, used bytes, available bytes.
- API Nodes query ZooKeeper for healthy disks (minimum 100MB available) when making placement decisions.
- Per-disk metrics visible in the Admin UI Dashboard.

