# Identity & Access Management (IAM)

ShannonStore provides an AWS IAM-compatible access control system.

- Users and groups can be managed through the Admin UI, with policies controlling what each user or group is allowed to do.
- Policies follow the standard AWS IAM JSON format, supporting wildcard matching and explicit Deny-over-Allow evaluation.
- Access key pairs can be generated for programmatic S3 API access, with support for expiration and status management.
- On first cluster start, a default admin user, group, and full-access policy are automatically created.
