# Encryption & Key Management (KMS)

ShannonStore provides enterprise-grade encryption with a distributed key management system.

## KMS Providers

- Cluster KMS (ClusterKmsProvider): Production implementation using RocksDB for key persistence and ZooKeeper for cluster-wide synchronization. Implements envelope encryption with
  AES-256-GCM.


## Envelope Encryption

- Data Encryption Key (DEK): A random 256-bit AES key generated per encryption operation.
- Key Encryption Key (KEK): Derived from the master password using PBKDF2WithHmacSHA256 (200,000 iterations, 32-byte salt).
- Encryption flow: Data is encrypted with a random DEK using AES/GCM/NoPadding (128-bit tag, 12-byte IV). The DEK is then encrypted (wrapped) with the active KEK. The output envelope
  contains: [version(4)] + [kekIV(12)] + [wrappedDekLen(4)] + [wrappedDek] + [dekIV(12)] + [encryptedData].
- Decryption flow: Parse envelope → look up KEK by version → unwrap DEK → decrypt data.

## Master Key Security

- The master key is stored in an environment variable (never in config files), specified by shannonstore.kms.master.key.env (default: SHANNONSTORE_MASTER_KEY).
- Derived using PBKDF2 with a random 32-byte salt and 200,000 iterations, producing a 256-bit KEK.

## Key Rotation

- Only the cluster leader can perform key rotations, preventing conflicts in distributed scenarios.
- Rotation creates a new KEK version while marking the old version as RETIRED (still available for decryption).
- Revoked keys prevent new encryption but maintain version history for existing data.

## Cluster Synchronization

- The leader exports the encrypted keystore and broadcasts it to follower nodes.
- Followers listen for update signals and reload from the leader with retry logic (configurable retry count and sleep interval).
- Replicas that cannot find a key version will attempt re-sync from RocksDB before failing.