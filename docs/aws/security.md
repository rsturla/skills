# AWS: Security

## Instance Metadata Service (IMDS)

Always enforce IMDSv2 (token-required). IMDSv1 is vulnerable to SSRF attacks that steal instance credentials.

```hcl
resource "aws_instance" "example" {
  metadata_options {
    http_endpoint               = "enabled"
    http_tokens                 = "required"
    http_put_response_hop_limit = 1
  }
}
```

- Set `http_tokens = "required"` on every EC2 instance, launch template, and ASG
- Set `http_put_response_hop_limit = 1` unless containers need metadata access (then use 2)
- Enforce account-wide:

```hcl
resource "aws_ec2_instance_metadata_defaults" "imdsv2" {
  http_tokens                 = "required"
  http_put_response_hop_limit = 1
}
```

If workloads use IAM Roles Anywhere or IRSA, disable IMDS entirely:

```hcl
metadata_options {
  http_endpoint = "disabled"
}
```

## IAM: No IAM Users

IAM users have long-lived credentials that leak, get committed to git, and never expire unless rotated.

- **Human access**: SAML federation with corporate IdP (Okta, Entra ID, etc.)
- **Service-to-service (within AWS)**: IAM roles — instance profiles, execution roles, IRSA
- **Service-to-service (external)**: Vault — dynamic credentials or AppRole authentication
- **CI/CD**: OIDC federation (GitHub Actions, GitLab CI) — no static credentials

If a legacy integration requires access keys: scope to minimum permissions, rotate every 90 days, store in Vault,
monitor via CloudTrail, and track a backlog item to migrate off.

## IAM Policies

- Least privilege always. Start with zero permissions, add what's needed
- Use AWS-managed policies only as starting points — scope down for production
- Avoid `*` in resource ARNs. Scope to specific resources or resource patterns
- Use conditions: `aws:RequestedRegion`, `aws:PrincipalOrgID`, `aws:SourceArn`
- Permission boundaries on roles created by developers/automation

## Tag-Based Access Control (ABAC)

Use tag conditions in IAM policies to scale permissions without per-resource ARN lists. ABAC lets you write
fewer policies that automatically apply to new resources with matching tags.

```json
{
  "Effect": "Allow",
  "Action": ["ec2:StartInstances", "ec2:StopInstances"],
  "Resource": "arn:aws:ec2:*:*:instance/*",
  "Condition": {
    "StringEquals": {
      "aws:ResourceTag/Owner": "${aws:PrincipalTag/Team}"
    }
  }
}
```

Key patterns:

- **`aws:ResourceTag/Key`**: match against tags on the target resource
- **`aws:PrincipalTag/Key`**: match against tags on the calling principal (role/user)
- **`aws:RequestTag/Key`**: enforce tags at resource creation time
- **`aws:TagKeys` + `ForAllValues`**: require specific tags are always present on new resources

Require tagging at creation (one statement per required tag — multiple `Null` keys in a single
condition use AND, so a combined check only denies when *all* tags are missing):

```json
[
  {
    "Effect": "Deny",
    "Action": ["ec2:RunInstances", "ec2:CreateVolume"],
    "Resource": "*",
    "Condition": {
      "Null": { "aws:RequestTag/Owner": "true" }
    }
  },
  {
    "Effect": "Deny",
    "Action": ["ec2:RunInstances", "ec2:CreateVolume"],
    "Resource": "*",
    "Condition": {
      "Null": { "aws:RequestTag/Environment": "true" }
    }
  }
]
```

Rules:

- Tag IAM roles with `Team`, `Environment`, and `Project` to enable principal-based matching
- Use ABAC for team-scoped access: teams manage only resources tagged with their team name
- Combine with resource-level ARN restrictions for defense in depth
- SAML session tags can pass team/role attributes from IdP directly into `aws:PrincipalTag`

## S3: Block Public Access

Every S3 bucket must have all four public access block settings enabled. No exceptions.

```hcl
resource "aws_s3_bucket_public_access_block" "example" {
  bucket = aws_s3_bucket.example.id

  block_public_acls       = true
  ignore_public_acls      = true
  block_public_policy     = true
  restrict_public_buckets = true
}
```

Account-level safety net:

```hcl
resource "aws_s3_account_public_access_block" "account" {
  block_public_acls       = true
  ignore_public_acls      = true
  block_public_policy     = true
  restrict_public_buckets = true
}
```

To serve static content, use CloudFront with Origin Access Control (OAC):

```hcl
resource "aws_cloudfront_origin_access_control" "oac" {
  name                              = "s3-oac"
  origin_access_control_origin_type = "s3"
  signing_behavior                  = "always"
  signing_protocol                  = "sigv4"
}
```

Bucket policy grants access only to CloudFront distribution ARN. Never use Origin Access Identity (OAI) — it is
legacy and does not support SSE-KMS or newer S3 features.

Deny HTTP transport:

```json
{
  "Effect": "Deny",
  "Action": "s3:*",
  "Resource": ["arn:aws:s3:::bucket", "arn:aws:s3:::bucket/*"],
  "Condition": {
    "Bool": { "aws:SecureTransport": "false" }
  },
  "Principal": "*"
}
```

## Encryption

- S3: SSE-S3 minimum, SSE-KMS for sensitive data
- EBS: encrypted by default (account-level setting)
- RDS: encryption at rest enabled at creation
- KMS: use customer-managed keys (CMKs) for data requiring key rotation control

## Logging and Monitoring

- **CloudTrail**: enabled in all regions, with log file validation (digest files) enabled
- **GuardDuty**: enabled — provides threat detection for compromised credentials, crypto mining,
  and anomalous API activity
- **Security Hub**: enabled — aggregates findings from GuardDuty, Config, and other sources
- **AWS Config**: enabled with conformance packs for compliance rules
- **VPC Flow Logs**: enabled on all VPCs, sent to CloudWatch or S3. Use v5 custom format with
  `vpc-id`, `subnet-id`, `pkt-srcaddr`, `pkt-dstaddr`, `flow-direction` fields for dual-stack
  troubleshooting
- **CloudWatch alarms**: billing thresholds, unauthorized API calls

## Network Security

- WAF on all public-facing ALBs and CloudFront distributions. Enable at minimum:
  `AWSManagedRulesCommonRuleSet`, `AWSManagedRulesSQLiRuleSet`, `AWSManagedRulesKnownBadInputsRuleSet`.
  Configure rate-limiting rules for API endpoints. Log WAF decisions to S3 or CloudWatch
- TLS 1.2+ everywhere. Pin specific policies:
  - ALB: `ELBSecurityPolicy-TLS13-1-2-2021-06` (not the default `2016-08` which allows TLS 1.0)
  - CloudFront: `TLSv1.2_2021` minimum protocol version
- Use ACM for certificate management
- No SSH/RDP to instances — use SSM Session Manager
