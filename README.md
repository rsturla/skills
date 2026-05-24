# skills

Agent skills for Claude Code, Cursor, and other coding agents.

## Install

```bash
# Global (available in all projects)
bunx skills add rsturla/skills -g

# Project-scoped
bunx skills add rsturla/skills

# Preview available skills
bunx skills add rsturla/skills --list
```

## Skills

| Skill | Description |
| ----- | ----------- |
| `vertex-ai` | Query and call Claude models on Vertex AI — model reference, endpoints, quick API calls |
| `commit-message` | Generate/improve git commit messages from staged changes via Vertex AI |
| `new-worktree` | Create git worktrees with conventional-commit directory structure, always relative to repo root |
| `write-a-skill` | Create new agent skills with proper structure and conventions |
| `openshift-troubleshoot` | Diagnose OpenShift issues — pod failures, SCCs, networking, storage, operators |
| `write-docs` | Write clear, structured technical documentation with consistent conventions |
| `investigate-cve` | Trace CVE to upstream fix commits, check backport status across branches |
| `dependency-audit` | Audit deps for vulnerabilities, license issues, and supply chain risks |
| `security-review` | Multi-agent security review — injection, auth, secrets, supply chain |
| `code-review` | Multi-agent code review — architecture, correctness, style, conventions |

## Recommended Third-Party Skills

```bash
bunx skills add JuliusBrussee/caveman -g
```

| Skill | Source | Description |
| ----- | ------ | ----------- |
| `caveman` | [JuliusBrussee/caveman](https://github.com/JuliusBrussee/caveman) | Ultra-compressed communication mode — cuts token usage ~75% |
| `caveman-review` | | Ultra-compressed code review comments |

## Coding Guidelines

This repo includes opinionated coding and platform guidelines in `docs/`. These are referenced by
`AGENTS.md` and automatically available to any agent configured with the symlinks below.

| Guideline | Description |
| --------- | ----------- |
| [Go](docs/GO_CODING_GUIDELINES.md) | Go conventions, error handling, project layout |
| [Rust](docs/RUST_CODING_GUIDELINES.md) | Rust idioms, error handling, crate structure |
| [Python](docs/PYTHON_CODING_GUIDELINES.md) | Python style, typing, project layout |
| [OpenTofu + Terragrunt](docs/OPENTOFU_CODING_GUIDELINES.md) | IaC conventions, module structure |
| [Containerfile](docs/CONTAINERFILE_GUIDELINES.md) | OCI image builds, multi-stage, security |
| [GitHub Actions](docs/GITHUB_ACTIONS_GUIDELINES.md) | CI/CD workflow conventions |
| [API Design](docs/API_DESIGN_GUIDELINES.md) | REST/gRPC API conventions |
| [AWS](docs/aws/README.md) | AWS platform guidelines — networking, compute, security, cost |

## Agent Configuration

`AGENTS.md` is the single source of truth for agent instructions. Symlink it (and the `docs/`
directory) into each agent's config directory so all agents share the same guidelines.

### Claude Code

```bash
ln -s /path/to/skills/AGENTS.md ~/.claude/CLAUDE.md
ln -s /path/to/skills/AGENTS.md ~/.claude/AGENTS.md
ln -s /path/to/skills/docs ~/.claude/docs
```

### Goose

```bash
ln -s /path/to/skills/AGENTS.md ~/.config/goose/AGENTS.md
ln -s /path/to/skills/docs ~/.config/goose/docs
```

### OpenCode

```bash
ln -s /path/to/skills/AGENTS.md ~/.config/opencode/AGENTS.md
ln -s /path/to/skills/docs ~/.config/opencode/docs
```

### Pi

```bash
ln -s /path/to/skills/AGENTS.md ~/.pi/agent/AGENTS.md
ln -s /path/to/skills/docs ~/.pi/agent/docs
```

Replace `/path/to/skills` with the absolute path to your clone of this repo (e.g.,
`/var/home/admin/Repos/rsturla/skills`).

All agents read through the symlinks at runtime — no copy or sync step needed. Changes to
`AGENTS.md` or any file in `docs/` are immediately available to every configured agent.

## Requirements

- `gcloud` CLI (authenticated)
- `jq`
- `curl`
- `oc` CLI (for OpenShift troubleshooting skill)
- Access to GCP project `itpc-gcp-core-pe-eng-claude`
