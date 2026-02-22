# Authentication & Authorization

## JWT-Based Authentication (Admin UI/API)

- Login returns an access token (JWT, 15-minute TTL) and a refresh token (UUID, 24-hour TTL).
- JWT signed with HMAC-SHA256, containing claims: subject (userId), isAdmin, requirePasswordChange, jti (unique token ID), iat, exp.
- Refresh tokens allow session extension without re-entering credentials.
- Explicit logout revokes the access token by tracking its jti until expiration.
- Background cleanup thread removes expired revoked tokens and refresh tokens every 3 hours.

## S3 API Authentication

- Parses AWS Signature V4 and V2 Authorization headers to extract the access key.
- Validates the access key against the in-memory index (status = Active, not expired).
- On validation failure, attempts a cluster-wide IAM reload before rejecting the request.

## Authorization Flow (S3 Requests)

1. Extract access key from the Authorization header.
2. Map the HTTP method + URI to an S3 action (e.g., PUT /bucket/key → s3:PutObject).
3. Construct the resource ARN: arn:aws:s3:::bucket/key.
4. Collect all applicable policies: user-attached policies + policies from all user's groups.
5. Evaluate using PolicyEvaluator: explicit Deny wins → then check for Allow → default Deny (ABSTAIN).
6. Return 403 Forbidden if not allowed; proceed with the operation if allowed.

