# AWS: Storage

## S3

- Block all public access at bucket and account level (see [Security](security.md))
- Enable versioning on all buckets
- Lifecycle rules: transition to IA/Glacier, expire old versions
- Encryption: SSE-S3 minimum, SSE-KMS for sensitive data
- Access logging to separate bucket

## EBS

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

## RDS / Aurora

- Prefer **Aurora Serverless v2** for variable workloads — scales to zero in dev, scales up in prod.
  Caveat: ~15s cold start on resume (longer after 24h idle), incompatible with RDS Proxy
- Use provisioned Aurora or RDS only for predictable, sustained throughput
- Multi-AZ for production, single-AZ acceptable for dev
- Encryption at rest enabled (cannot be added after creation)
- No public accessibility — internal subnets only
- IAM database authentication where supported
- Automated backups with appropriate retention (minimum 7 days prod)
- Performance Insights enabled for troubleshooting

## DynamoDB

- Default choice for key-value and document workloads
- **On-demand capacity** for unpredictable traffic, provisioned for steady-state
- Enable point-in-time recovery (PITR)
- Encryption at rest with AWS-owned key (default) or CMK for compliance
- Use DynamoDB Streams + Lambda for event-driven patterns
