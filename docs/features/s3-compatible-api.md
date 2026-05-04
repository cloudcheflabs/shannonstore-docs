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
- **ETag**: Returned on PUT, GET, HEAD, ListObjects, UploadPart, and CompleteMultipartUpload responses. Single-PUT objects use the MD5 of the plaintext body; multipart-completed objects use the standard `MD5(concatenation of part MD5s) + "-N"` composite form, so AWS SDKs and `aws s3 cp` can verify object integrity.
- **Content-MD5 Verification**: When a client provides the `Content-MD5` request header on PUT or UploadPart, the server recomputes MD5 of the received body and rejects the write with `400 BadDigest` on mismatch.

## Two-port design

Object storage and admin/management traffic run on separate ports (defaults 8080 and 8888), so admin tooling and cluster operations never compete with the data path.
