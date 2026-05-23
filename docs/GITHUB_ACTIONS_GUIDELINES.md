# GitHub Actions Guidelines

## Action Pinning

All actions MUST be pinned by **full SHA digest** at the **latest available version**. Never use floating tags like
`@v4` or `@main`.

```yaml
# Good — pinned by digest with version comment
- uses: actions/checkout@de0fac2e4500dabe0009e67214ff5f5447ce83dd # v6

# Bad — floating tag, vulnerable to supply chain attacks
- uses: actions/checkout@v4
```

Why: tag-based references can be force-pushed. Digest pins are immutable and prevent supply chain attacks.

## Step Names

Every step MUST have a `name` field. No anonymous steps.

```yaml
# Good
- name: Checkout repository
  uses: actions/checkout@de0fac2e4500dabe0009e67214ff5f5447ce83dd # v6

# Bad — no name
- uses: actions/checkout@de0fac2e4500dabe0009e67214ff5f5447ce83dd # v6
```

## Workflow Structure

```yaml
name: CI

on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read

jobs:
  lint:
    runs-on: ubuntu-24.04
    steps:
      - name: Checkout repository
        uses: actions/checkout@de0fac2e4500dabe0009e67214ff5f5447ce83dd # v6
```

## Security

- Set **minimal permissions** at workflow level with `permissions:`
- Never use `permissions: write-all` or leave permissions unset (defaults to write-all in some orgs)
- Use `pull_request` not `pull_request_target` unless you need write access to the base repo
- Never echo secrets into logs
- Use `GITHUB_TOKEN` over PATs where possible
- Pin runner images: `runs-on: ubuntu-24.04` not `ubuntu-latest` for reproducibility

## Updating Actions

When updating an action version:

1. Find the latest release tag
2. Get the full commit SHA for that tag
3. Update both the SHA and the version comment

```bash
# Get digest for a specific tag
gh api repos/actions/checkout/git/ref/tags/v4.2.2 --jq '.object.sha'
```

## Gotchas

- `ubuntu-latest` can change OS version without notice — pin to `ubuntu-24.04` for reproducibility
- `actions/setup-node` caches are scoped per branch — first PR run is always slower
- `GITHUB_TOKEN` permissions differ between `pull_request` and `push` triggers
- Composite actions don't support `secrets` context — pass secrets as inputs
- `concurrency` with `cancel-in-progress: true` can kill deploy jobs mid-flight — only use for CI, not CD

## Anti-Patterns

- Floating action tags (`@v4`, `@main`)
- Anonymous steps (no `name:`)
- `permissions: write-all`
- Hardcoded secrets in workflow files
- `ubuntu-latest` for reproducible builds
- Long monolithic workflows — split into reusable workflows or composite actions
