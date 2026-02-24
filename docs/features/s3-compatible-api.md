# S3-Compatible API

ShannonStore provides a fully compatible Amazon S3 REST API, allowing applications to use standard S3 SDKs and tools (AWS CLI, boto3, MinIO client, etc.) without any code changes.

## Supported Operations

- **Object Operations**: PutObject, GetObject, HeadObject, DeleteObject — full CRUD lifecycle with support for key prefixes simulating directory structures.
- **Bucket Operations**: CreateBucket, DeleteBucket, ListBuckets, ListObjects — bucket management with AWS-compatible XML responses.
- **Multipart Upload**: Upload large objects in parallel parts, assembled on completion.
- **Object Versioning**: Per-bucket versioning with unique version IDs, version history, and delete markers.
- **Range Requests**: Partial object downloads for efficient large file access and resumable downloads.
- **AWS Authentication**: Supports both AWS Signature V4 and legacy V2 authentication.
- **Chunked Transfer Encoding**: Supports AWS SDK streaming upload format.

## Server Architecture

The S3 API runs on a custom NIO HTTP server with a two-port design:
- **S3 Port** (default 8080): Optimized for high-throughput object storage operations.
- **Admin Port** (default 8888): Serves the Admin UI, REST API, and cluster management endpoints.
