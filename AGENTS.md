# Coding Style

## General

- No trailing whitespace
- Files end with a single newline
- UTF-8 encoding everywhere
- Lines max 120 chars (code), 120 chars (prose/markdown)

## Naming

- Files and directories: kebab-case (`new-worktree`, `commit-message`)
- Branches: `dev/robertsturla/<change>` with kebab-case change name
- Commits: Conventional Commits format, subject ≤50 chars, body lines ≤72 chars

## Shell (bash)

- `set -uo pipefail` (avoid `set -e` in scripts with arithmetic or grep)
- Use `[[` not `[` for conditionals
- Quote all variable expansions: `"$var"` not `$var`
- Heredocs for multi-line strings, not nested quotes
- Functions before main logic
- Prefer `command -v` over `which`

## Markdown (SKILL.md)

- YAML frontmatter: `name`, `description`, `allowed-tools`, `metadata`
- One blank line between sections
- Fenced code blocks with language tag
- Tables for structured reference data
- No emoji unless explicitly requested

## Git

- Conventional Commits: `feat`/`fix`/`refactor`/`docs`/`test`/`chore`/`ci`/`build`/`perf`/`style`
- Imperative mood in subject line, lowercase after prefix (`feat: add thing` not `feat: Add thing`)
- No trailing period on subject
- Body explains why, not what
- **Atomic commits** — one logical change per commit, each commit should build and pass tests independently
- Never amend published commits
- To fix a previous commit, use `git commit --fixup=<sha>` then `git rebase -i --autosquash` before pushing
- Prefer fixup over amend — keeps history clean while preserving the ability to review intermediate states

## Language Guides

- [Go Coding Guidelines](docs/GO_CODING_GUIDELINES.md)
- [Rust Coding Guidelines](docs/RUST_CODING_GUIDELINES.md)
- [Python Coding Guidelines](docs/PYTHON_CODING_GUIDELINES.md)
- [OpenTofu + Terragrunt Guidelines](docs/OPENTOFU_CODING_GUIDELINES.md)
- [Containerfile Guidelines](docs/CONTAINERFILE_GUIDELINES.md)
- [GitHub Actions Guidelines](docs/GITHUB_ACTIONS_GUIDELINES.md)
- [API Design Guidelines](docs/API_DESIGN_GUIDELINES.md)

## Tool & Language Preferences

- **Containers**: podman, never docker. Use `Containerfile`, never `Dockerfile`
- **VCS**: git
- **Languages** (order of preference): Go → Rust → Python
- **IaC**: OpenTofu + Terragrunt, never Terraform
- **JS/TS runtime**: Bun, never npm/yarn/pnpm
- **LLM APIs**: Vertex AI only (GCP project `itpc-gcp-core-pe-eng-claude`). No direct Anthropic API, OpenAI, Copilot,
  or other LLM provider access. Always use the Vertex AI rawPredict endpoint. See the `vertex-ai` skill for details.
- **Shell**: bash for scripts
- **Package installs**: NEVER run any system package manager (`dnf`, `brew`, `apt`, `yum`, `pacman`, `zypper`, etc.).
  If a tool is missing, ask the user to provide it.

## Dependencies

- **In applications**: use proper language SDKs (Go, Rust, Python). Do NOT shell out to `curl`/`jq` from application code.
- **In skills and git hooks**: use standard unix tools (`curl`, `jq`, `gcloud`, `oc`) since skills are markdown
  instructions and hooks should avoid runtime dependencies.
