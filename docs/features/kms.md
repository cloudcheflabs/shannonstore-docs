# Encryption & Key Management (KMS)

ShannonStore provides server-side encryption with a built-in distributed key management system.

- Uses envelope encryption: data is encrypted with a per-object key, which is itself encrypted by a master-derived key (AES-256-GCM).
- The master key is stored as an environment variable and never written to config files.
- Key rotation is supported — new keys are created for future encryption while old keys remain available for decrypting existing data.
- Key state is automatically synchronized across all cluster nodes by the leader.
