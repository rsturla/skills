# AWS Guidelines

These guidelines assume a tenant perspective within a managed AWS Organization. Account vending, SCPs, and
org-level controls are managed by the platform team.

## Contents

- [IaC and Tagging](iac-and-tagging.md) — OpenTofu/CloudFormation, mandatory tags, provider defaults
- [Networking](networking.md) — Dual-stack VPCs, 3-tier subnets, NAT strategy, VPC endpoints, DNS
- [Compute](compute.md) — Serverless-first (Lambda → OpenShift → EC2), instance types, Graviton
- [Storage](storage.md) — S3, EBS volume selection, RDS/Aurora Serverless, DynamoDB
- [Security](security.md) — IMDSv2, IAM (no users, ABAC), S3 hardening, encryption, WAF, TLS
- [Operations](operations.md) — Containers, secrets, DR/backup, certs, quotas, cost, observability

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
