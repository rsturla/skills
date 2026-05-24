# AWS: Operations

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

## Observability

- OpenTelemetry for instrumentation (see global OTel preference)
- ADOT Collector or OTel Collector sidecar for export
- X-Ray for distributed tracing when OTel integration is not yet available
- Structured JSON logging — never unstructured text
- CloudWatch metrics for AWS-native resources, OTLP export for application metrics
