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

## Recommended Third-Party Skills

```bash
bunx skills add JuliusBrussee/caveman -g
```

| Skill | Source | Description |
| ----- | ------ | ----------- |
| `caveman` | [JuliusBrussee/caveman](https://github.com/JuliusBrussee/caveman) | Ultra-compressed communication mode — cuts token usage ~75% |
| `caveman-review` | | Ultra-compressed code review comments |

## Requirements

- `gcloud` CLI (authenticated)
- `jq`
- `curl`
- `oc` CLI (for OpenShift troubleshooting skill)
- Access to GCP project `itpc-gcp-core-pe-eng-claude`
