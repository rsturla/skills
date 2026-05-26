---
name: new-worktree
description: >
  Create a git worktree with conventional-commit directory structure and standardized branch naming.
  Always creates relative to the repo root, not the current directory. Use when the user says
  "new worktree", "create worktree", "start working on", "new branch", or invokes /new-worktree —
  even if they just say "set up a branch for this work" or "I need to work on a fix."
compatibility: Requires git and git-worktree-add custom wrapper.
allowed-tools:
  - Bash(git *)
metadata:
  author: rsturla
  version: 1.0.0
---

# New Worktree

Create worktrees sorted by conventional commit type at the repo root, with branch naming
`dev/robertsturla/<change>`.

## Usage

```text
/new-worktree <type> <change>
```

- `type`: conventional commit type — `feat`, `fix`, `chore`, `docs`, `refactor`, `perf`, `test`, `build`, `ci`,
  `style`, `revert`
- `change`: kebab-case name describing the work

## Behavior

1. **Parse args** — extract `<type>` and `<change>` from user input. If missing, ask.
2. **Validate type** — must be one of: `feat`, `fix`, `chore`, `docs`, `refactor`, `perf`, `test`, `build`, `ci`,
   `style`, `revert`
3. **Validate change** — must be kebab-case (`[a-z0-9]+(-[a-z0-9]+)*`). If user provides spaces or other formats,
   auto-convert to kebab-case and confirm.
4. **Find repo root** — works for both bare and normal repos:

   ```bash
   REPO_ROOT=$(realpath "$(git rev-parse --git-common-dir)/..")
   ```

   `git-common-dir` returns `.bare` (bare repo) or `.git` (normal repo). Parent of either is the repo root.
5. **Detect default branch** — find remote HEAD, don't hardcode `main`:

   ```bash
   DEFAULT_BRANCH=$(git symbolic-ref refs/remotes/origin/HEAD | sed 's|refs/remotes/origin/||')
   ```

6. **Fetch latest default branch** — ensure worktree starts from up-to-date ref, not stale local:

   ```bash
   git fetch origin "$DEFAULT_BRANCH"
   ```

7. **Create worktree** — always relative to repo root, based on `origin/<default>`:

   ```bash
   git worktree-add "${REPO_ROOT}/<type>/<change>" -b dev/robertsturla/<change> "origin/${DEFAULT_BRANCH}"
   ```

8. **Report** — show created path and branch name.

## Examples

User: `/new-worktree feat create-monitoring-service`

```bash
REPO_ROOT=$(realpath "$(git rev-parse --git-common-dir)/..")
DEFAULT_BRANCH=$(git symbolic-ref refs/remotes/origin/HEAD | sed 's|refs/remotes/origin/||')
git fetch origin "$DEFAULT_BRANCH"
git worktree-add "${REPO_ROOT}/feat/create-monitoring-service" -b dev/robertsturla/create-monitoring-service "origin/${DEFAULT_BRANCH}"
```

Result: worktree at `<repo-root>/feat/create-monitoring-service`, branch
`dev/robertsturla/create-monitoring-service`

User: `/new-worktree fix resolve-auth-timeout`

```bash
git fetch origin "$DEFAULT_BRANCH"
git worktree-add "${REPO_ROOT}/fix/resolve-auth-timeout" -b dev/robertsturla/resolve-auth-timeout "origin/${DEFAULT_BRANCH}"
```

User: `/new-worktree chore update-dependencies`

```bash
git fetch origin "$DEFAULT_BRANCH"
git worktree-add "${REPO_ROOT}/chore/update-dependencies" -b dev/robertsturla/update-dependencies "origin/${DEFAULT_BRANCH}"
```

## Gotchas

- Use `git worktree-add` (custom wrapper), **NOT** `git worktree add`. The wrapper writes relative `.git` paths for
  portability.
- `git rev-parse --show-toplevel` returns wrong path in bare repos. Always use
  `realpath "$(git rev-parse --git-common-dir)/.."` to find repo root.
- If a worktree or branch already exists, report the conflict and ask the user — don't overwrite.
- If `git symbolic-ref refs/remotes/origin/HEAD` fails (no `origin/HEAD` set), run
  `git remote set-head origin --auto` first, then retry.
- Branch always prefixed `dev/robertsturla/`. Change name used for both directory suffix and branch suffix — they
  always match.
