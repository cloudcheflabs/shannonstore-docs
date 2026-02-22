# Configuration Management

## Priority Chain (highest to lowest)

1. System Properties: JVM -Dkey=value arguments (highest priority).
2. Environment Variables: Auto-converted from property key format (dots → underscores, uppercase).
3. External Config File: Specified via shannonstore.config.file property or SHANNONSTORE_CONFIG_FILE environment variable.
4. Default Classpath Properties: shannonstore.properties bundled in the JAR (lowest priority).

## Variable Resolution

- Supports ${variable} syntax in property values for DRY configuration (e.g., ${shannonstore.base.data.dir}/s3-metadata).
- Recursive resolution across all configuration sources.

150+ Configuration Parameters covering: ZooKeeper connection, S3/Admin ports, storage paths, thread pool sizes, EC shard counts, KMS settings, JWT token TTLs, RocksDB tuning, metrics
collection intervals, disk repair thresholds, compression settings, and more.

