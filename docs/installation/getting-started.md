# Getting Started

This walks through your first end-to-end interaction with a freshly-installed ShannonStore cluster: signing in to the admin UI, creating an S3 bucket and access key, and exercising the bucket from the AWS CLI. If the cluster isn't running yet, start with [Download &amp; Install](installation.md).

## Open the Admin UI

Once `bin/start-example-servers.sh` reports the cluster is ready, point a browser at:

```agsl
http://localhost:8888/admin
```

The first sign-in uses the bootstrap credentials `admin` / `admin`. The UI immediately requires a password change before any other action is permitted.

<img width="1200" src="../../images/getting-started/dashboard.png"/>

The dashboard shows the leader API server, registered Data Nodes, per-node ownership distribution, KMS state, and per-disk capacity.

## Create a Bucket and an Access Key

To upload objects you need two things: an S3 bucket and a credential pair.

**Create an access key.** In the **Identity Control** menu, click **New Keypair**. A new keypair is generated and stored encrypted under the cluster's KMS. Click the download icon to save the access key and secret to a JSON file — the secret is only shown once.

<img width="1200" src="../../images/getting-started/credential.png"/>

**Create a bucket.** In the **Object Store** menu, click **New Bucket** and enter `test-bucket` as the name.

> Buckets and IAM state replicate to every API Server in the cluster via runtime sync. There is no per-server bucket — once a bucket is created on the leader, every node can serve it.

## Configure the AWS CLI

ShannonStore speaks SigV4. Configure a profile with the keypair you just downloaded:

```agsl
aws configure --profile=custom-s3 set default.s3.signature_version s3v4
aws configure --profile=custom-s3 set aws_access_key_id <access-key>
aws configure --profile=custom-s3 set aws_secret_access_key <secret-key>
aws configure --profile=custom-s3 set region us-east-1
```

The S3 endpoint for the local cluster is `http://localhost:8080`. Pass it as `--endpoint-url` on every command (or set it once via `aws configure --profile=custom-s3 set endpoint_url`).

## Upload, List, Download, Delete

```agsl
# Upload an object
aws s3 cp ./values-dev_old.yaml s3://test-bucket/values-dev_old.yaml \
  --profile=custom-s3 \
  --endpoint-url http://localhost:8080 \
  --no-verify-ssl
```

```agsl
# List the bucket
aws s3 ls s3://test-bucket \
  --profile=custom-s3 \
  --endpoint=http://localhost:8080 \
  --no-verify-ssl \
  --recursive --human-readable --summarize
```

```agsl
# Download
aws s3 cp s3://test-bucket/values-dev_old.yaml values-dev_downloaded.yaml \
  --profile=custom-s3 \
  --endpoint=http://localhost:8080
```

```agsl
# Delete
aws s3 rm s3://test-bucket/values-dev_old.yaml \
  --profile=custom-s3 \
  --endpoint=http://localhost:8080
```

The objects you uploaded are erasure-coded into 4+2 (configurable) shards on upload, and reconstructed transparently on download — even if one Data Node or one disk is unavailable.

## What's Next

- Learn about the [S3-compatible API](../features/s3-compatible-api.md) — supported S3 operations, multipart upload, ETag/MD5 semantics.
- See how data survives failures in [Erasure Coding](../features/ec.md) and [Disk Repair Service](../features/disk-repair.md).
- Configure encryption modes in [Encryption &amp; Key Management](../features/kms.md).
- Run a real deployment — see [Cluster Operations](../operations/operations.md) for starting/stopping individual roles, monitoring, and maintenance procedures.

## Stopping the Example Cluster

```agsl
bin/stop-example-servers.sh
```
