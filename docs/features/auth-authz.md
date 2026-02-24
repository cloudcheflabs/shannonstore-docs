# Authentication & Authorization

## Admin UI/API Authentication

- JWT-based authentication with short-lived access tokens and longer-lived refresh tokens.
- Supports explicit logout with token revocation.

## S3 API Authentication

- Parses AWS Signature V4 and V2 headers to extract and validate access keys.
- On validation failure, attempts a cluster-wide IAM reload before rejecting the request.

## S3 Authorization Flow

- Each S3 request is mapped to an action and resource ARN, then evaluated against all applicable IAM policies (user-attached + group-inherited).
- Explicit Deny always wins; if no Allow is found, access is denied by default.
