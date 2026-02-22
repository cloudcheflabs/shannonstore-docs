# Identity & Access Management (IAM)

ShannonStore implements a full IAM system compatible with AWS IAM policy format for fine-grained access control.

## User Management

- Create, read, update, and delete users with unique user IDs.
- Password-based authentication for Admin UI access.
- Force password change on first login (requirePasswordChange flag).
- Users can belong to multiple groups simultaneously.

## Group Management

- Organize users into groups for collective permission management.
- Policies attached to groups automatically apply to all members.
- Default admin-group created on initialization with AdministratorAccess policy.

## Policy System (AWS-Compatible)

- Policies use the AWS IAM JSON format with Version, Statement, Effect (Allow/Deny), Action, and Resource fields.
- Wildcard matching: Supports * (any substring) and ? (single character) in both Action and Resource fields. Example: s3:Get* matches s3:GetObject and s3:GetBucketLocation.
- ARN format: Resources use arn:aws:s3:::bucket-name/key-prefix/* format.
- Explicit Deny: Deny statements always take precedence over Allow statements, matching AWS IAM behavior.
- Pattern matching uses regex caching with LRU eviction (max 10,000 patterns) for performance.

## Policy Attachment

- Policies can be attached directly to users (attach-user-policy) or to groups (attach-group-policy).
- During authorization, all applicable policies (user-direct + group-inherited) are collected and evaluated together.

## Access Key Management

- Generate access key pairs (16-character access key ID + 32-character secret key) for programmatic S3 API access.
- Access keys have optional expiration dates and Active/Inactive status.
- Credentials downloadable as CSV files from the Admin UI.
- Access keys are indexed in memory (ConcurrentHashMap) for fast O(1) lookup during S3 request authorization.

## Default Initialization

- On first cluster start, the system creates: AdministratorAccess policy (allows all actions on all resources), admin-group with this policy attached, and admin user in the admin group
  with forced password change.