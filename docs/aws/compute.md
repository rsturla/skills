# AWS: Compute

**Serverless first.** Prefer managed/serverless options. Only use EC2 or self-managed compute when serverless
does not meet requirements (GPU, sustained high throughput, stateful workloads, licensing).

Decision order:

```text
Lambda → OpenShift (platform-managed) → EC2
```

## Lambda (Preferred)

- Default choice for event-driven, API, and short-lived workloads
- **ARM64 (`arm64`)** architecture always — better cost and performance. Note: native binary
  dependencies (C extensions, shared libraries) and Lambda layers must be compiled for ARM
- Minimum permissions via execution role
- Set `reserved_concurrent_executions` to prevent runaway invocations
- Use Lambda Powertools for structured logging and tracing
- Use provisioned concurrency only for latency-sensitive paths
- Prefer function URLs or API Gateway over ALB for HTTP triggers

## OpenShift (Platform-Managed)

- Default choice for long-running processes, containerised workloads, and services needing orchestration
- Use platform-managed OpenShift clusters (ROSA or team-provided)
- Do not self-manage EKS clusters — use platform-provided OpenShift
- Pod Identity / IRSA for pod-level IAM — never share node instance profile
- Follow OpenShift security context constraints (SCCs) over raw Kubernetes PSPs
- See [OpenShift Troubleshooting skill](../../skills/openshift-troubleshoot/) for operational guidance

## EC2 (Last Resort)

- Only for workloads that cannot run serverless: GPU, licensed software, stateful services, sustained CPU
- IMDSv2 enforced (see [Security](security.md))
- Always use launch templates, never launch configurations
- EBS volumes: gp3 default, encrypted always

## EC2 Instance Type Selection

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

## Other Serverless Services

Prefer managed/serverless equivalents across the board:

| Instead of | Use |
| --- | --- |
| Self-managed message broker | SQS, SNS, EventBridge |
| Self-managed API proxy | API Gateway |
| Cron on EC2 | EventBridge Scheduler + Lambda |
| Self-managed step workflows | Step Functions |
| Self-managed database | DynamoDB (serverless), Aurora Serverless |
