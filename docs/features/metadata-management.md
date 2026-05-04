# Metadata Management

ShannonStore manages object metadata via a vbucket-on-HRW model with per-shard RocksDB sub-stores.

## Placement model

- Every key is hashed to one of **65,536 fixed vbuckets** (Murmur3 mod 65,536).
- Each vbucket's **top-R API nodes** by HRW (Rendezvous Hashing) score are its owners; the highest-scoring is the primary, the rest are replicas. Default `R = 2` (`shannonstore.api.metadata.replication.factor`).
- Per-key ownership lookup is O(1) — the vbucket→owner table is cached and only invalidated when the live API node set changes.
- Adding or removing a node moves only ~1/N of vbuckets (HRW invariant), and every key inside a moved vbucket moves together.

## Storage layout (per API node)

- Metadata lives in **N RocksDB sub-stores**, one per shard, where each shard owns a contiguous vbucket range. Default `N = 16` (`shannonstore.api.metadata.rocksdb.shards`); 65,536 must be evenly divisible.
- Shards can be **striped across multiple disks** via `shannonstore.api.metadata.rocksdb.dirs=/disk1,/disk2,...` — shard `i` lands on `dirs[i % len]/shard-i/`, so compaction and IO are isolated per disk.
- An in-memory LRU caches the most recently-read entries (`shannonstore.api.metadata.latest.cache.size`).

## Replication

- Default mode: **`PULL`** (`shannonstore.api.metadata.replication.mode`). The primary commits locally and acks the client; replicas pull new entries every 500 ms (`shannonstore.api.metadata.pull.interval.ms`) using a per-peer cursor and HRW-scoped filtering.
- Optional modes: `SYNC` (primary waits for every replica ACK) and `ASYNC_PUSH` (primary fans out off the hot path).
- New or restarting nodes pull a one-shot **bootstrap snapshot** from every peer after `cluster-ready`, so they start with everything they own under HRW.

## Membership change

- Node added/removed → vbucket ownership shifts → new owners pull missing keys via the PULL cycle (or push fan-out in SYNC/ASYNC_PUSH).
- A periodic **reconciliation sweep** (off by default; `shannonstore.api.metadata.reconcile.enabled`) drops local copies that this node is no longer an owner of, after verifying the current primary holds the same-or-newer copy. Cassandra's `nodetool cleanup` equivalent.

## Read path

- Local hit → return.
- Local miss + I am an owner: in `SYNC` mode return null (every owner committed before the client ack); in `PULL`/`ASYNC_PUSH` modes fall back to the primary so a client GET that nginx routed to a lagging owner doesn't see a spurious 404.
- Local miss + I am not an owner → fetch from the primary via the metadata fetch RPC.
