---
name: scaffold-repo
description: >
  Bootstrap new repositories with standard tooling — CI pipelines, linting, Renovate, gitleaks, CLAUDE.md, and
  language-specific project structure for Go, Rust, and Python. Supports GitHub Actions and GitLab CI. Use when
  the user wants to set up a new project, scaffold a repo, bootstrap a codebase, or initialize a repository —
  even if they just say "start a new project", "set up a Go service", or "init a Rust crate."
compatibility: Requires git. Optional gh CLI for GitHub, glab CLI for GitLab.
allowed-tools:
  - Bash(git *)
  - Bash(gh *)
  - Bash(glab *)
  - Bash(ls *)
  - Bash(find *)
  - Bash(ln -s *)
  - Bash(sed *)
  - Read
  - Write
  - Edit
metadata:
  author: rsturla
  version: 1.0.0
---

# Repository Scaffolding

Bootstrap a new repo with CI, linting, dependency management, secret scanning, and language-specific project structure.

## Quick Start

```text
User: /scaffold-repo
Agent: asks project name, language, repo type, forge
Agent: generates all files, presents list for review
User: approves or requests changes
```

## Process

1. Check prerequisites (git init, existing files)
2. Gather inputs interactively (project name, language, repo type, forge)
3. Auto-detect forge from git remote
4. Generate all files — do NOT commit
5. Present complete file list for user review
6. Iterate on feedback until user approves
7. Suggest follow-up skills

## Prerequisites

Before scaffolding, check if the directory is a git repository:

```bash
if [[ ! -d .git ]]; then
    # Ask user: "No git repo found. Run git init?"
    git init
fi
```

If the repo already has files (non-empty beyond `.git/`), ask user whether this is:

- **Fresh scaffold** — full generation, warn about conflicts
- **Additive scaffold** — add specific components only (see Partial Scaffolding)

## Step 1 — Gather Inputs

Ask these questions in order (later answers depend on earlier ones):

1. **Project name** — kebab-case, used for module name and directory structure
2. **Language** — Go, Rust, or Python
3. **Repo type** — Service, CLI, or Library
4. **Forge** — auto-detect, then confirm
5. **License** — MIT, Apache-2.0, or none (default: MIT)

### Python Package Name

Python package directories cannot contain hyphens. Convert `<project_name>` to `<package_name>` by replacing
hyphens with underscores (e.g., `my-service` becomes `my_service`). Use `<package_name>` for all Python directory
and import paths.

### Forge Detection

```bash
REMOTE_URL=$(git remote get-url origin 2>/dev/null || echo "")
if [[ "$REMOTE_URL" == *github.com* ]]; then
    FORGE="github"
elif [[ "$REMOTE_URL" == *gitlab* ]]; then
    FORGE="gitlab"
fi
```

Always confirm detected forge with user. If no remote, ask directly.

## Step 2 — Generate Files

Read templates from [REFERENCE.md](REFERENCE.md), substitute placeholders, write files.

| Category | Files | Varies By |
| -------- | ----- | --------- |
| Project | `CLAUDE.md`, `AGENTS.md` (symlink), `.gitignore`, `.editorconfig`, `CODEOWNERS` | Language |
| License | `LICENSE` | User choice |
| CI | Workflow/pipeline files | Forge + Language |
| Dependencies | Renovate config | Forge |
| Security | `.gitleaks.toml`, `.pre-commit-config.yaml` | — |
| Language | Module config, linter config, project layout | Language + Type |

### Placeholders

| Placeholder | Source | Example |
| ----------- | ------ | ------- |
| `<project_name>` | User input (kebab-case) | `my-service` |
| `<package_name>` | `<project_name>` with hyphens → underscores (Python only) | `my_service` |
| `<module_path>` | Git remote or user input | `github.com/rsturla/my-service` |
| `<description>` | User input | `REST API for widget management` |
| `<year>` | Current year | `2026` |

### File Generation Rules

- Read each template from REFERENCE.md
- Replace all placeholders listed above
- Use `Write` tool for each file
- Create AGENTS.md symlink: `ln -s CLAUDE.md AGENTS.md`
- **Never commit** — present file list for review first
- **Never overwrite** existing files without asking

### Forge-Specific Paths

| File | GitHub | GitLab |
| ---- | ------ | ------ |
| CI pipeline | `.github/workflows/ci.yml` | `.gitlab-ci.yml` |
| Dependency review | `.github/workflows/dependency-review.yml` | — |
| Renovate | `.github/renovate.json` | `.gitlab/renovate.json` |

## Step 3 — Review and Iterate

After generating all files:

1. List all created files with `find . -type f -not -path './.git/*' | sort`
2. Present summary: "Created N files. Review and let me know if you want changes."
3. Wait for explicit approval before any git operations
4. If user requests changes, modify and re-present

### Follow-Up Suggestions

After user approves scaffold, suggest:

- `/write-docs` — generate a proper README.md
- `/dependency-audit` — initial dependency vulnerability check
- `/commit-message` — write the initial commit message

## File Customization Rules

### By Repo Type

| Type | Entry Point | Build Step | Notes |
| ---- | ----------- | ---------- | ----- |
| Service | Go: `cmd/<name>/main.go`, Rust: `src/main.rs`, Python: `src/<name>/__main__.py` | Yes | Includes `internal/` (Go) |
| CLI | Same as Service | Yes | Go: consider cobra for subcommands |
| Library | Go: root package, Rust: `src/lib.rs`, Python: `src/<name>/` | No | CI: test + lint only |

### Type-Specific .gitignore Adjustments

- **Service/CLI**: commit lockfiles (`uv.lock`, `Cargo.lock`) for reproducible builds
- **Library**: gitignore lockfiles (`uv.lock`, `Cargo.lock`) — consumers manage their own

### Module Path Inference

For Go, infer module path from git remote:

```bash
REMOTE_URL=$(git remote get-url origin 2>/dev/null || echo "")
if [[ -n "$REMOTE_URL" ]]; then
    MODULE_PATH=$(echo "$REMOTE_URL" | sed -E 's|.*[:/](.+)(\.git)?$|\1|')
fi
```

If no remote or ambiguous, ask user directly. Note: GitLab nested groups produce paths with more than three
components (e.g., `gitlab.com/org/group/subgroup/repo`) — the regex handles this correctly.

## Partial Scaffolding

When adding components to an existing repo, ask user which categories to generate:

- **CI only** — generate workflow/pipeline files
- **Dependencies only** — Renovate config
- **Security only** — gitleaks + pre-commit
- **Language tooling only** — linter config, project layout

Skip the full interactive flow. Only ask questions relevant to selected categories.

## Gotchas

- `AGENTS.md` is a **symlink pointing to** `CLAUDE.md` — CLAUDE.md is the real file
- Renovate config uses **JSONC format** with `.json` extension — Renovate parses JSONC from `.json` files natively.
  Always use `best-practices` preset
- GitHub Actions must be **pinned by full SHA digest** — templates use `<sha>` placeholders. Always resolve to
  current SHAs at generation time using `gh api`. See REFERENCE.md "Resolving Action SHAs" section. Never copy
  stale SHAs from templates without verifying
- Go module path: infer from git remote or ask. Never hardcode
- Python: always use **uv**, never pip/poetry/pipenv. Package directories use underscores, not hyphens
- Rust: use `edition = "2024"` and `resolver = "3"`
- `.gitignore` must be **language-specific** — a Go .gitignore is very different from Python. Lockfile handling
  differs by repo type (commit for services, ignore for libraries)
- Pre-commit hooks reference gitleaks — tool must be installed separately. Skill generates config only
- Check for existing files before writing. If conflicts, ask user: overwrite, skip, or merge
- CODEOWNERS syntax differs between GitHub (`@org/team`) and personal repos (`@username`) — ask user

See [REFERENCE.md](REFERENCE.md) for all templates.
