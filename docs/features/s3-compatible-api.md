# S3-Compatible API

ShannonStore provides a fully compatible Amazon S3 REST API, allowing applications to interact with the storage system using standard S3 SDKs and tools (AWS CLI, boto3, MinIO client, etc.)
without any code changes.

## Supported Operations

- Object Operations: PutObject, GetObject, HeadObject, DeleteObject — full CRUD lifecycle for objects with support for arbitrary key prefixes simulating directory structures.
- Bucket Operations: CreateBucket, DeleteBucket, ListBuckets, ListObjects — bucket-level management with XML response format matching AWS S3 specification (ListBucketResult,
  ListAllMyBucketsResult).
- Multipart Upload: InitiateMultipartUpload, UploadPart, CompleteMultipartUpload, AbortMultipartUpload — enables uploading large objects in parts. Each upload is assigned a unique uploadId
  with node affinity. Parts can be uploaded in parallel and are assembled on completion.
- Object Versioning: Per-bucket versioning can be enabled via PutBucketVersioning. When enabled, each PutObject generates a unique versionId (UUID). Previous versions are preserved and
  accessible via ListObjectVersions. Deleting a versioned object creates a delete marker rather than removing data.
- Range Requests: HTTP Range header support for partial object downloads (bytes=start-end, bytes=-suffix). Returns 206 Partial Content with Content-Range header, enabling efficient large
  file access and resumable downloads.
- AWS Chunked Transfer Encoding: Supports content-encoding: aws-chunked and x-amz-content-sha256: STREAMING-* headers used by AWS SDKs for streaming uploads. The AwsChunkedInputStream
  decoder transparently unwraps the chunked framing (hex size + chunk-signature metadata) to extract the raw object data.
- AWS Signature V4 Authentication: Parses the Authorization header in both AWS Signature V4 (AWS4-HMAC-SHA256 Credential=...) and legacy V2 (AWS accessKey:signature) formats to extract the
  access key for IAM policy evaluation.

## NIO HTTP Server Architecture

The S3 API runs on a custom-built NIO HTTP/1.1 server (NioHttpServer) designed for high throughput. A selector thread accepts connections in non-blocking mode, reads HTTP headers, then
switches the channel to blocking mode and dispatches to a configurable worker thread pool (default: CPU cores × multiplier). TCP_NODELAY is enabled for low-latency responses, and socket
send/receive buffers are tuned (default 2MB each). The S3RequestHandler routes requests by HTTP method and URI pattern to the appropriate handler, performing IAM authorization before
processing.