# OpenTofu + Terragrunt Coding Guidelines

## Project Structure

Terragrunt 1.0 uses a flatter structure with units and stacks. Shared `.tf` modules are referenced via `source`,
not colocated with `terragrunt.hcl`.

```text
infrastructure/
  modules/                  # Reusable .tf modules (versioned separately)
    vpc/
      main.tf, variables.tf, outputs.tf, versions.tf
    rds/
    eks/
  terragrunt.hcl            # Root config (remote state, provider generate)
  _envcommon/               # Shared Terragrunt includes per component
    vpc.hcl
    rds.hcl
  dev/
    env.hcl                 # account_id, env name
    us-east-1/
      region.hcl
      vpc/terragrunt.hcl    # unit — references modules/vpc via source
      rds/terragrunt.hcl    # unit — references modules/rds via source
  staging/
  prod/
    terragrunt.stack.hcl    # explicit stack — templates entire environment
```

- Each unit's `terragrunt.hcl` references shared `.tf` code via `terraform { source = "..." }`
- No `.tf` files in unit directories — only `terragrunt.hcl`
- Explicit stacks (`terragrunt.stack.hcl`) template entire environments from reusable patterns
- Separate state per environment and per component
- Differences between environments driven by input variables, never by forking HCL
- Keep modules under 500 lines

## Naming Conventions

- **Identifiers**: `_` (underscore), not `-` (dash). Lowercase only.
- **Resources**: don't repeat resource type in name. The **primary resource** in a module is always named `this`. Use
  descriptive names (`public`, `private`, `client`) for secondary resources.
- **Variables**: mirror provider argument names. Plural for `list`/`map` types. Always include `description`.
- **Outputs**: `{name}_{type}_{attribute}` — e.g. `private_subnet_ids`, `security_group_id`
- **Files**: `main.tf`, `variables.tf`, `outputs.tf`, `versions.tf` (required). Split by component only when files
  exceed ~200 lines.
- **Env vars**: use `OPENTOFU_VAR_` prefix, not `TF_VAR_`

```hcl
# Good
resource "aws_route_table" "public" {}

# Bad
resource "aws_route_table" "public_route_table" {}
```

## Module Design

### Two-Layer Pattern

- **Resource modules**: wrap a single resource type with sensible defaults
- **Service modules**: compose resource modules into deployable units

### Primary Resource is `this`

The main resource in a module is always named `this`. Supporting resources get descriptive names.

```hcl
# modules/rds/main.tf
resource "aws_db_instance" "this" { ... }           # primary
resource "aws_security_group" "this" { ... }         # primary SG for the instance
resource "aws_security_group" "client" { ... }       # client SG for consumers
resource "aws_db_subnet_group" "this" { ... }        # supporting
```

### Prefer `name_prefix` Over Explicit Names

Use `name_prefix` (or equivalent `_prefix` arguments) instead of hardcoded `name` values. This lets AWS generate
unique names and avoids conflicts across environments.

```hcl
# Good — unique names, no conflicts
resource "aws_security_group" "this" {
  name_prefix = "${var.name}-rds-"
  vpc_id      = var.vpc_id
}

# Bad — hardcoded name, breaks in multi-env
resource "aws_security_group" "this" {
  name   = "rds-security-group"
  vpc_id = var.vpc_id
}
```

This applies to security groups, IAM roles, launch templates, and any resource supporting `name_prefix`.

### Dependency Inversion

Modules receive dependencies as inputs — never create their own. An `rds` module accepts `vpc_id` and `subnet_ids`,
never creates a VPC.

### Client Security Group Pattern

Modules that provide a network service should create a **client security group** that consumers attach to their own
resources. This inverts the dependency — the service module owns the access rules, consumers just reference the group.

```hcl
# modules/rds/main.tf
resource "aws_security_group" "this" {
  name_prefix = "${var.name}-rds-"
  vpc_id      = var.vpc_id
}

resource "aws_security_group" "client" {
  name_prefix = "${var.name}-rds-client-"
  vpc_id      = var.vpc_id
}

resource "aws_security_group_rule" "allow_clients" {
  type                     = "ingress"
  from_port                = 5432
  to_port                  = 5432
  protocol                 = "tcp"
  security_group_id        = aws_security_group.this.id
  source_security_group_id = aws_security_group.client.id
}

# modules/rds/outputs.tf
output "client_security_group_id" {
  description = "Attach this SG to resources that need database access"
  value       = aws_security_group.client.id
}
```

```hcl
# Consumers just attach the client SG — no ingress rules needed
resource "aws_instance" "app" {
  vpc_security_group_ids = [
    module.rds.client_security_group_id,
    aws_security_group.app.id,
  ]
}
```

Benefits:

- Service module owns its access policy — single source of truth
- Consumers don't need to know ports, protocols, or CIDR blocks
- Adding/removing a consumer never touches the service module
- Works for RDS, ElastiCache, OpenSearch, any service with network access

### Input/Output Contracts

Variables and outputs form the module's API. Use `precondition`/`postcondition` blocks to make assumptions explicit.
Parameterize sparingly — expose only values that genuinely vary between deployments.

## Terragrunt Patterns (1.0+)

Terragrunt 1.0 (March 2026) introduced breaking changes. All examples use 1.0 syntax.

### Terminology

- **Unit**: a single directory with `terragrunt.hcl` — one deployable piece of infrastructure
- **Stack**: a collection of units managed together, defined via `terragrunt.stack.hcl`

### CLI Syntax (1.0)

```bash
# Run a command across all units
terragrunt run --all -- plan
terragrunt run --all -- apply

# Single unit
terragrunt run -- plan

# Filter by path or dependency
terragrunt run --all --filter "path:dev/*" -- plan
```

`run-all`, `plan-all`, `apply-all` are **removed**. All `--terragrunt-*` flags shortened (e.g. `--non-interactive`).

### DRY Remote State

Include account ID in bucket name to prevent cross-account conflicts. Define once in root config:

```hcl
remote_state {
  backend = "s3"
  config = {
    bucket         = "${local.account_id}-tofu-state"
    key            = "${path_relative_to_include()}/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "${local.account_id}-tofu-locks"
  }
}
```

### Dependencies

- `dependency` (singular): when you need outputs from another module
- `dependencies` (plural): when you only need execution ordering
- Always provide `mock_outputs` so `validate` and `plan` work before dependencies are applied
- Dependencies only expose **outputs**, not inputs (changed in 1.0)
- External dependencies (outside CWD) are **excluded by default** — use `--queue-include-external` if needed

### Generate Blocks

Use to inject `provider.tf`, `backend.tf`, or any boilerplate. Even alone, `generate` eliminates dozens of identical
files.

### Error Handling (1.0)

`retryable_errors` is replaced by `errors` block. `skip` is replaced by `exclude`.

## State Management

- **Remote backend**: always. S3 + DynamoDB (or native S3 locking in OpenTofu 1.10+)
- **Per-environment isolation**: separate state per env and major component
- **State encryption**: enable OpenTofu's native client-side encryption. Use KMS for prod, PBKDF2 for dev.
- **State locking**: always enabled. Never disable. Stuck locks = investigate concurrent runs.
- **Never**: commit state to git, edit state JSON manually, share state across environments

## Testing

Testing pyramid (fast → slow):

1. `tofu fmt` — formatting check (every commit)
2. `tofu validate` — syntax and type checking
3. Static analysis (Checkov, tfsec) — security scanning
4. `tofu test` with `command = plan` — unit tests against plan output
5. `tofu test` with `command = apply` — integration tests (CI)
6. Terratest — full end-to-end (pre-merge for critical changes only)

Always commit `.terraform.lock.hcl`.

## Security

- **Credentials**: never hardcode. Use IAM roles, OIDC federation, workload identity.
- **Sensitive variables**: mark `sensitive = true`. Better: write secrets directly to a secrets manager, never as
  OpenTofu outputs.
- **Ephemeral resources** (1.11+): use for short-lived tokens that should never persist to state.
- **Policy as code**: Checkov/tfsec in CI against plan output. Block non-compliant applies.
- **Supply chain**: pin provider and module versions. Commit lock file. Verify checksums.

## Anti-Patterns

- Secrets in state outputs
- Monolithic state (one file for everything)
- God modules (VPC + EKS + RDS in one module)
- Thin wrappers around single resources with no added value
- `latest` tags in providers/modules
- Manual state edits — use `tofu state mv`, `tofu import`
- Circular Terragrunt dependencies — redesign module graph
- Hardcoded values — use variables with defaults
