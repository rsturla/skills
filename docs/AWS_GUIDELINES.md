# AWS Guidelines

These guidelines assume a tenant perspective within a managed AWS Organization. Account vending, SCPs, and
org-level controls are managed by the platform team.

## IaC and Tooling

- **OpenTofu + Terragrunt** preferred, never Terraform (licensing)
- CloudFormation acceptable when OpenTofu is not viable (AWS-native integrations, org constraints)
- Pin provider versions. Use a version constraint like `~> 5.0`
- Store state in S3 with DynamoDB locking. Enable versioning on state bucket
- One state file per environment per component — avoid monolithic statefiles
- Use `terragrunt.hcl` for DRY configuration across environments
- Tag all resources at the provider level with default tags:

```hcl
provider "aws" {
  region = var.region

  default_tags {
    tags = {
      app-code      = var.app_code
      service-phase = var.environment
      cost-center   = var.cost_center
      ManagedBy     = "opentofu"
      Owner         = var.team
    }
  }
}
```

## Service Decisions

When choosing between AWS services or external alternatives, use these defaults:

| Want to use... | Use instead | Why |
| --- | --- | --- |
| Terraform | OpenTofu + Terragrunt (or CloudFormation) | Licensing; OpenTofu preferred, CloudFormation acceptable |
| EKS (self-managed) | OpenShift (platform-managed) | Platform team manages control plane |
| ECR | Quay | Platform-managed, better scanning, multi-cloud |
| Docker | Podman | Daemonless, rootless, OCI-native |
| Secrets Manager / Parameter Store | Vault (platform-managed) | Dynamic secrets, rotation, cross-platform |
| IAM Users / access keys | SAML federation + IAM roles | No long-lived credentials |
| NAT Gateway (dev) | NAT instance (`t4g.nano`) | ~80% cheaper for low-traffic dev |
| gp2 EBS | gp3 EBS | ~20% cheaper, better baseline, tunable IOPS/throughput |
| io1 EBS | io2 EBS | Better durability, same cost |
| x86 instances | Graviton (ARM) | ~20% better price-performance |
| Origin Access Identity (OAI) | Origin Access Control (OAC) | OAI is legacy, no SSE-KMS support |
| Public S3 bucket | CloudFront + OAC | S3 public access blocked, always |
| SSH / RDP | SSM Session Manager | No open ports, audited sessions |
| Self-managed certs | ACM (DNS validation) | Auto-renewal, no manual rotation |
| ACM PCA | Vault PKI | ACM PCA expensive; Vault already available |
| Datadog SDK | OpenTelemetry | Vendor-neutral observability |
| Reserved Instances | Compute Savings Plans | Covers Lambda + EC2, more flexible |

## Tagging

### Mandatory Organisation Tags

At least 80% of cloud resources within an account must carry these tags. Non-compliance is tracked
and reported:

| Tag | Value | Description |
| --- | --- | --- |
| `app-code` | CMDB Application ID | Links resource to application in CMDB |
| `service-phase` | `dev`, `stage`, `prod` | Deployment environment |
| `cost-center` | 3-digit code | Cost Center funding the resource |

### Additional Team Tags

Beyond org-mandatory tags, apply these for operational visibility:

| Tag | Purpose |
| --- | --- |
| `ManagedBy` | `opentofu`, `cloudformation`, `manual` |
| `Owner` | Team or individual responsible |

Tag-based cost allocation and access control depend on consistent tagging. Use AWS Config rules to detect
untagged resources.

## Networking

### VPC Design

- One VPC per environment per region. Avoid default VPC for any workload
- **Dual-stack (IPv4 + IPv6)**: enable IPv6 on all VPCs and subnets where possible
- CIDR blocks: plan for growth, avoid overlaps across accounts (use RFC 1918 ranges for IPv4)
- **Avoid common home/office network CIDRs** — VPN and remote workers will have routing conflicts.
  Do not use: `192.168.0.0/16`, `10.0.0.0/24`, `10.0.1.0/24`, `172.16.0.0/16`. Prefer ranges
  deep in `10.x.x.x` space (e.g., `10.64.0.0/16`, `10.128.0.0/16`) that are unlikely to collide
  with home routers or corporate LANs
- Request Amazon-provided IPv6 CIDR (`/56`) per VPC
- Minimum three AZs for production workloads
- Three-tier subnet architecture:

```text
Public   → ALB, NAT Gateway/Instance, API Gateway endpoints
           Routes: 0.0.0.0/0 → IGW, ::/0 → IGW
Private  → Lambda, OpenShift (with NAT for IPv4 outbound, egress-only IGW for IPv6)
           Routes: 0.0.0.0/0 → NAT GW/Instance, ::/0 → Egress-only IGW
Internal → RDS, ElastiCache, internal services (no route to internet)
           Routes: no default route, VPC endpoints only
```

```hcl
resource "aws_vpc" "main" {
  cidr_block                       = "10.0.0.0/16"
  assign_generated_ipv6_cidr_block = true
  enable_dns_hostnames             = true
  enable_dns_support               = true
}

resource "aws_subnet" "private" {
  vpc_id                          = aws_vpc.main.id
  cidr_block                      = "10.0.1.0/24"
  ipv6_cidr_block                 = cidrsubnet(aws_vpc.main.ipv6_cidr_block, 8, 1)
  assign_ipv6_address_on_creation = true
}
```

### NAT Strategy

- **Production**: NAT Gateway (managed, HA per AZ)
- **Dev / Sandbox**: NAT instance (`t4g.nano` or `t4g.micro`) to reduce cost. NAT Gateway charges per GB
  processed — dev environments with low traffic save significantly with NAT instances

```hcl
resource "aws_instance" "nat" {
  ami                    = data.aws_ami.nat_instance.id
  instance_type          = "t4g.nano"
  source_dest_check      = false
  subnet_id              = aws_subnet.public.id

  metadata_options {
    http_endpoint = "enabled"
    http_tokens   = "required"
  }
}
```

### Routing

- Workloads in private subnets. Only load balancers and NAT in public subnets
- Use **egress-only internet gateway** for IPv6 outbound from private subnets (NAT is IPv4-only)
- Use VPC endpoints (gateway for S3/DynamoDB, interface for other services) to avoid NAT costs and keep traffic
  on AWS backbone. Apply endpoint policies to restrict access to specific bucket ARNs and account IDs —
  without policies, any principal in the VPC can reach any bucket/table
- Transit Gateway for multi-VPC/multi-account connectivity. Avoid VPC peering mesh. Note: TGW has
  per-attachment and per-GB data processing charges — evaluate whether simple VPC peering is
  sufficient for two-VPC setups

### Security Groups and NACLs

- Security groups: principle of least privilege
- No `0.0.0.0/0` or `::/0` ingress except public ALBs on 443
- Reference security groups by ID, not CIDR, for internal traffic
- NACLs: use as coarse-grained backup. Default deny inbound on internal subnets.
  Always mirror NACL rules for both `::/0` and `0.0.0.0/0` — a common mistake is adding only
  IPv4 deny rules and leaving IPv6 open
- Include both IPv4 and IPv6 rules in security groups
- **IPv6 addresses are globally routable.** Unlike IPv4 behind NAT, there is no implicit ingress
  protection from address translation. Ensure security groups explicitly deny unwanted IPv6 ingress

### DNS

- Route 53 private hosted zones for internal service discovery
- Use alias records for AWS resources (ALB, CloudFront, S3) — no CNAME at zone apex
- Enable AAAA records for dual-stack endpoints

## Compute

**Serverless first.** Prefer managed/serverless options. Only use EC2 or self-managed compute when serverless
does not meet requirements (GPU, sustained high throughput, stateful workloads, licensing).

Decision order:

```text
Lambda → OpenShift (platform-managed) → EC2
```

### Lambda (Preferred)

- Default choice for event-driven, API, and short-lived workloads
- **ARM64 (`arm64`)** architecture always — better cost and performance. Note: native binary
  dependencies (C extensions, shared libraries) and Lambda layers must be compiled for ARM
- Minimum permissions via execution role
- Set `reserved_concurrent_executions` to prevent runaway invocations
- Use Lambda Powertools for structured logging and tracing
- Use provisioned concurrency only for latency-sensitive paths
- Prefer function URLs or API Gateway over ALB for HTTP triggers

### OpenShift (Platform-Managed)

- Default choice for long-running processes, containerised workloads, and services needing orchestration
- Use platform-managed OpenShift clusters (ROSA or team-provided)
- Do not self-manage EKS clusters — use platform-provided OpenShift
- Pod Identity / IRSA for pod-level IAM — never share node instance profile
- Follow OpenShift security context constraints (SCCs) over raw Kubernetes PSPs
- See [OpenShift Troubleshooting skill](../skills/openshift-troubleshoot/) for operational guidance

### EC2 (Last Resort)

- Only for workloads that cannot run serverless: GPU, licensed software, stateful services, sustained CPU
- IMDSv2 enforced (see Security section)
- Always use launch templates, never launch configurations
- EBS volumes: gp3 default, encrypted always

### EC2 Instance Type Selection

**ARM (Graviton) first.** Graviton instances offer ~20% better price-performance over x86. Only use x86 when
software has no ARM support.

| Family | Use Case | Notes |
| --- | --- | --- |
| `t4g` | Dev/test, low-traffic, bursty | ARM. Burstable, cheapest. Watch CPU credits |
| `m7g` / `m8g` | General purpose, steady-state | ARM. Default production choice |
| `c7g` / `c8g` | Compute-intensive (builds, encoding) | ARM. High CPU-to-memory ratio |
| `r7g` / `r8g` | Memory-intensive (caches, in-memory DB) | ARM. High memory-to-CPU ratio |
| `m7i` / `c7i` | x86-only software | x86. Use only when ARM unavailable |
| `g5` / `p5` | GPU (ML inference, rendering) | x86/NVIDIA. No ARM GPU option |

Rules:

- Always use latest generation available in region
- Never use previous-gen (`m5`, `c5`, `t3`) for new workloads
- Right-size using CloudWatch metrics or Compute Optimizer before committing to Savings Plans
- Use Spot instances for fault-tolerant batch workloads (70-90% savings)

### Other Serverless Services

Prefer managed/serverless equivalents across the board:

| Instead of | Use |
| --- | --- |
| Self-managed message broker | SQS, SNS, EventBridge |
| Self-managed API proxy | API Gateway |
| Cron on EC2 | EventBridge Scheduler + Lambda |
| Self-managed step workflows | Step Functions |
| Self-managed database | DynamoDB (serverless), Aurora Serverless |

## Storage

### S3

- Block all public access at bucket and account level (see Security section)
- Enable versioning on all buckets
- Lifecycle rules: transition to IA/Glacier, expire old versions
- Encryption: SSE-S3 minimum, SSE-KMS for sensitive data
- Access logging to separate bucket

### EBS

- Always encrypted — enable account-level default:

```hcl
resource "aws_ebs_encryption_by_default" "enabled" {
  enabled = true
}
```

Choose volume type based on workload:

| Type | Use Case | Notes |
| --- | --- | --- |
| `gp3` | **Default.** General purpose | 3000 IOPS / 125 MiB/s baseline, independently tunable |
| `io2` / `io2 Block Express` | High-performance databases, latency-sensitive | Provision IOPS only when benchmarks justify cost |
| `st1` | Throughput-intensive sequential (logs, data lakes) | Cannot be boot volume. 500 MiB/s max |
| `sc1` | Cold storage, infrequent access | Cheapest. Cannot be boot volume |

Rules:

- **gp3 over gp2** always — ~20% cheaper per GiB, better baseline performance, tunable IOPS/throughput
- Never use `gp2` or `io1` for new workloads
- Snapshot lifecycle policies for backups via AWS Backup or DLM

### RDS / Aurora

- Prefer **Aurora Serverless v2** for variable workloads — scales to zero in dev, scales up in prod.
  Caveat: ~15s cold start on resume (longer after 24h idle), incompatible with RDS Proxy
- Use provisioned Aurora or RDS only for predictable, sustained throughput
- Multi-AZ for production, single-AZ acceptable for dev
- Encryption at rest enabled (cannot be added after creation)
- No public accessibility — internal subnets only
- IAM database authentication where supported
- Automated backups with appropriate retention (minimum 7 days prod)
- Performance Insights enabled for troubleshooting

### DynamoDB

- Default choice for key-value and document workloads
- **On-demand capacity** for unpredictable traffic, provisioned for steady-state
- Enable point-in-time recovery (PITR)
- Encryption at rest with AWS-owned key (default) or CMK for compliance
- Use DynamoDB Streams + Lambda for event-driven patterns

## Container Registry

**Prefer Quay** (platform-managed) over ECR for container image storage.

- Push images to Quay. Use ECR only when AWS service integration requires it (Lambda container images,
  App Runner, etc.)
- Scan images on push — Quay's Clair scanner or equivalent
- Lifecycle policies to expire untagged images after 7 days
- Never use `:latest` tag in production — use immutable digest-based or semver tags
- Sign production images (cosign or Quay's built-in signing)
- Multi-arch images (ARM + x86) for Graviton compatibility

## Secrets Management

**Prefer platform-managed Vault** (HashiCorp Vault) over AWS Secrets Manager or Parameter Store.

- Store all secrets (database credentials, API keys, certificates) in Vault
- Use Vault's dynamic secrets for database credentials where possible — no long-lived DB passwords
- Rotate secrets automatically via Vault policies
- Application code reads from Vault at runtime — never bake secrets into images or environment variables
- Naming convention: `secret/{env}/{service}/{secret-name}` (e.g., `secret/prod/payments/stripe-api-key`)

### When to Use AWS-Native Secrets

Use AWS Secrets Manager or SSM Parameter Store only when:

- AWS service requires it (RDS password rotation via native integration)
- Vault is not available in the environment
- Static configuration that happens to be sensitive (license keys) → SSM Parameter Store SecureString

Never store secrets in environment variables, config files, or source code.

## Disaster Recovery and Backup

### DR Tiers

Classify every workload into a DR tier. Default is backup-restore.

| Tier | RTO | RPO | Strategy |
| --- | --- | --- | --- |
| Backup-restore | Hours | 24h | Restore from backups |
| Pilot-light | 30 min | Minutes | Minimal infra running, scale up on failover |
| Warm-standby | Minutes | Seconds | Scaled-down copy running in DR region |
| Active-active | Seconds | Near-zero | Full capacity in multiple regions |

### AWS Backup

- Use AWS Backup for all supported resources (EBS, RDS, DynamoDB, EFS, S3)
- Enable vault lock (WORM) for production backup vaults
- Cross-region backup copies for critical workloads
- Test restores quarterly — untested backups are not backups

### Backup Retention

| Environment | Minimum Retention |
| --- | --- |
| Dev | 7 days |
| PreProd | 14 days |
| Prod | 30 days (90 days for compliance-sensitive data) |

## Certificate Management (ACM)

- **ACM for all public TLS certificates** — no self-managed certs on ALBs or CloudFront
- Use **DNS validation** exclusively (not email). ACM auto-renews DNS-validated certs
- Wildcard certs (`*.example.com`) acceptable for internal services, specific certs for public-facing
- For internal/mTLS between services, use Vault PKI engine — not ACM

## Service Quotas

- Request quota increases **proactively** before go-live for: Lambda concurrency, API Gateway throttling,
  EC2 vCPU limits, NAT Gateway bandwidth, EBS volume limits
- Baseline request: 2x expected peak
- Set CloudWatch alarms on usage vs quota at 80% utilization
- Review quotas as part of production readiness checklist

## Cost Management

### Savings Plans

- **Compute Savings Plans** over Reserved Instances — cover Lambda and EC2 with flexibility
- Right-size for 3-6 months of usage data before committing
- RIs only for stable database workloads (RDS, ElastiCache) — Savings Plans do not cover RDS/ElastiCache

### Cost Controls

- Set AWS Budgets alerts at 80% and 100% of forecast per account
- Enable Cost Allocation Tags (align with required tagging)
- Monthly cost review per team — use Cost Explorer grouped by `Owner` tag
- Dev/sandbox accounts: aggressive cost controls (NAT instances, smaller instance types, Aurora Serverless
  scale-to-zero)

### Cost-Saving Patterns

| Pattern | Savings |
| --- | --- |
| Graviton (ARM) instances | ~20% over x86 |
| Spot instances (batch/fault-tolerant) | 70-90% |
| NAT instance in dev (vs NAT Gateway) | ~80% for low traffic |
| gp3 over gp2 | ~20% cheaper, better baseline |
| Aurora Serverless v2 in dev | Scale to zero when idle |
| VPC endpoints for S3/DynamoDB | Avoid NAT Gateway data processing charges |

## Security

### Instance Metadata Service (IMDS)

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

### IAM: No IAM Users

IAM users have long-lived credentials that leak, get committed to git, and never expire unless rotated.

- **Human access**: SAML federation with corporate IdP (Okta, Entra ID, etc.)
- **Service-to-service (within AWS)**: IAM roles — instance profiles, execution roles, IRSA
- **Service-to-service (external)**: Vault — dynamic credentials or AppRole authentication
- **CI/CD**: OIDC federation (GitHub Actions, GitLab CI) — no static credentials

If a legacy integration requires access keys: scope to minimum permissions, rotate every 90 days, store in Vault,
monitor via CloudTrail, and track a backlog item to migrate off.

### IAM Policies

- Least privilege always. Start with zero permissions, add what's needed
- Use AWS-managed policies only as starting points — scope down for production
- Avoid `*` in resource ARNs. Scope to specific resources or resource patterns
- Use conditions: `aws:RequestedRegion`, `aws:PrincipalOrgID`, `aws:SourceArn`
- Permission boundaries on roles created by developers/automation

### Tag-Based Access Control (ABAC)

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

### S3: Block Public Access

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

### Encryption

- S3: SSE-S3 minimum, SSE-KMS for sensitive data
- EBS: encrypted by default (account-level setting)
- RDS: encryption at rest enabled at creation
- KMS: use customer-managed keys (CMKs) for data requiring key rotation control

### Logging and Monitoring

- **CloudTrail**: enabled in all regions, with log file validation (digest files) enabled
- **GuardDuty**: enabled — provides threat detection for compromised credentials, crypto mining,
  and anomalous API activity
- **Security Hub**: enabled — aggregates findings from GuardDuty, Config, and other sources
- **AWS Config**: enabled with conformance packs for compliance rules
- **VPC Flow Logs**: enabled on all VPCs, sent to CloudWatch or S3. Use v5 custom format with
  `vpc-id`, `subnet-id`, `pkt-srcaddr`, `pkt-dstaddr`, `flow-direction` fields for dual-stack
  troubleshooting
- **CloudWatch alarms**: billing thresholds, unauthorized API calls

### Network Security

- WAF on all public-facing ALBs and CloudFront distributions. Enable at minimum:
  `AWSManagedRulesCommonRuleSet`, `AWSManagedRulesSQLiRuleSet`, `AWSManagedRulesKnownBadInputsRuleSet`.
  Configure rate-limiting rules for API endpoints. Log WAF decisions to S3 or CloudWatch
- TLS 1.2+ everywhere. Pin specific policies:
  - ALB: `ELBSecurityPolicy-TLS13-1-2-2021-06` (not the default `2016-08` which allows TLS 1.0)
  - CloudFront: `TLSv1.2_2021` minimum protocol version
- Use ACM for certificate management
- No SSH/RDP to instances — use SSM Session Manager

## Observability

- OpenTelemetry for instrumentation (see global OTel preference)
- ADOT Collector or OTel Collector sidecar for export
- X-Ray for distributed tracing when OTel integration is not yet available
- Structured JSON logging — never unstructured text
- CloudWatch metrics for AWS-native resources, OTLP export for application metrics

## Quick Reference

| Topic | Rule | Enforcement |
| --- | --- | --- |
| IaC | OpenTofu + Terragrunt preferred, CloudFormation acceptable | CI/CD pipeline |
| Compute | Serverless first: Lambda → OpenShift → EC2 | Architecture review |
| Instance types | ARM (Graviton) first, latest generation | Compute Optimizer + review |
| Networking | Dual-stack IPv4+IPv6, 3-tier subnets | VPC module + review |
| NAT | NAT Gateway prod, NAT instance dev/sandbox | Cost review |
| Tagging | 4 required tags on all resources | AWS Config rules |
| IMDS | IMDSv2 only, hop limit 1 | Account default |
| IAM users | None — use SAML/roles/OIDC | Policy + review |
| IAM policies | Least privilege, no `*` resources | Permission boundaries + review |
| IAM ABAC | Tag-based access control, team-scoped | SAML session tags + policy conditions |
| S3 public access | Blocked at bucket and account level | Account-level block |
| S3 static hosting | CloudFront + OAC, never public bucket | PR review + policy |
| EBS | gp3 default, always encrypted | Account default encryption |
| Encryption | Enabled everywhere (S3, EBS, RDS) | Account defaults |
| Secrets | Vault preferred, AWS-native only when required | Architecture review |
| Container registry | Quay preferred, ECR only for AWS-native integration | Pipeline config |
| Certificates | ACM with DNS validation, Vault PKI for internal | ACM + Vault |
| Backups | AWS Backup, test restores quarterly | Backup policies |
| Cost | Compute Savings Plans, budget alerts, Graviton | AWS Budgets |
| Logging | CloudTrail, Config, VPC Flow Logs | Account setup |
| Network security | WAF, no SSH, TLS 1.2+, SSM Session Manager | SG rules + policy |
| Observability | OpenTelemetry, structured JSON logs | Instrumentation standards |
| Service quotas | Proactive increases, alarms at 80% | Production readiness |
