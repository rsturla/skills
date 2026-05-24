# AWS: IaC and Tagging

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
