# AWS: Networking

## VPC Design

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

## NAT Strategy

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

## Routing

- Workloads in private subnets. Only load balancers and NAT in public subnets
- Use **egress-only internet gateway** for IPv6 outbound from private subnets (NAT is IPv4-only)
- Use VPC endpoints (gateway for S3/DynamoDB, interface for other services) to avoid NAT costs and keep traffic
  on AWS backbone. Apply endpoint policies to restrict access to specific bucket ARNs and account IDs —
  without policies, any principal in the VPC can reach any bucket/table
- Transit Gateway for multi-VPC/multi-account connectivity. Avoid VPC peering mesh. Note: TGW has
  per-attachment and per-GB data processing charges — evaluate whether simple VPC peering is
  sufficient for two-VPC setups

## Security Groups and NACLs

- Security groups: principle of least privilege
- No `0.0.0.0/0` or `::/0` ingress except public ALBs on 443
- Reference security groups by ID, not CIDR, for internal traffic
- NACLs: use as coarse-grained backup. Default deny inbound on internal subnets.
  Always mirror NACL rules for both `::/0` and `0.0.0.0/0` — a common mistake is adding only
  IPv4 deny rules and leaving IPv6 open
- Include both IPv4 and IPv6 rules in security groups
- **IPv6 addresses are globally routable.** Unlike IPv4 behind NAT, there is no implicit ingress
  protection from address translation. Ensure security groups explicitly deny unwanted IPv6 ingress

## DNS

- Route 53 private hosted zones for internal service discovery
- Use alias records for AWS resources (ALB, CloudFront, S3) — no CNAME at zone apex
- Enable AAAA records for dual-stack endpoints
