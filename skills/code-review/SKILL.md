---
name: code-review
description: >
  Comprehensive code review using multiple specialized agents. Checks architecture, error handling, naming, testing,
  and style against project conventions. Use when the user asks for a code review, says "review this", "check my
  code", or wants feedback on changes — even if they just say "what do you think?" about a diff.
compatibility: Requires git. Optional gh CLI for GitHub PRs, glab CLI for GitLab MRs.
allowed-tools:
  - Bash(git *)
  - Bash(gh *)
  - Bash(glab *)
  - Bash(grep *)
  - Bash(find *)
  - Read
  - Agent
metadata:
  author: rsturla
  version: 1.0.0
---

# Code Review

Multi-agent code review that produces actionable findings aligned with project conventions.

## Process

1. Detect review target (staged changes, branch diff, or PR)
2. Identify language and load relevant coding guidelines
3. Launch 3 specialized agents in parallel
4. Consolidate findings into a single report

## Step 1: Detect Review Target

```bash
# Check for staged changes first
if [ -n "$(git diff --cached --name-only)" ]; then
    DIFF_CMD="git diff --cached"
    TARGET="staged changes"
# Check for GitHub PR
elif gh pr view --json number &>/dev/null 2>&1; then
    DIFF_CMD="gh pr diff"
    TARGET="PR #$(gh pr view --json number -q .number)"
# Check for GitLab MR
elif glab mr view &>/dev/null 2>&1; then
    DIFF_CMD="glab mr diff"
    TARGET="MR !$(glab mr view --output json 2>/dev/null | jq -r .iid)"
# Fall back to branch diff vs main
else
    BASE=$(git merge-base HEAD origin/main 2>/dev/null || echo "origin/main")
    DIFF_CMD="git diff $BASE...HEAD"
    TARGET="branch $(git branch --show-current) vs main"
fi
```

## Step 2: Identify Language

Detect from changed files:

- `.go` → apply Go Coding Guidelines
- `.rs` → apply Rust Coding Guidelines
- `.py` → apply Python Coding Guidelines
- `.tf`, `.hcl` → apply OpenTofu + Terragrunt Guidelines
- `Containerfile` → apply Containerfile Guidelines
- `.yml`, `.yaml` in `.github/workflows/` → apply GitHub Actions Guidelines

Read the relevant guidelines doc before launching agents. Pass the guidelines content to each agent.

## Step 3: Launch Specialized Agents

Launch **3 agents in parallel** using the Agent tool. Each gets the full diff, changed files, and relevant guidelines.

### Agent 1: Architecture & Design

Prompt:

> Review this diff for architectural issues. Check for:
>
> - Interface design: are interfaces defined at the consumer? Are they small (1-3 methods)?
> - Dependency direction: do modules depend on abstractions, not concrete implementations?
> - Package/module boundaries: is the change in the right place? Should it be extracted?
> - Error handling: are errors wrapped with context? Are they handled at the right level?
> - Missing abstractions: should a new interface be introduced for pluggability?
> - Over-engineering: unnecessary abstractions, premature generalization, dead code
>
> Reference these project conventions: `<insert relevant guidelines>`
> For each finding: file, line, issue, suggestion.

### Agent 2: Correctness & Edge Cases

Prompt:

> Review this diff for bugs, correctness issues, and missing edge cases. Check for:
>
> - Nil/null handling: unchecked returns, nil pointer dereferences, Option unwraps
> - Concurrency: race conditions, missing locks, goroutine leaks, unsynchronized shared state
> - Resource leaks: unclosed files, connections, HTTP bodies
> - Off-by-one errors in loops, slices, pagination
> - Missing error paths: what happens when this fails?
> - Incomplete migrations: renamed function but callers not updated, partial refactors
>
> For each finding: file, line, bug description, severity (BUG/RISK/NITPICK), and fix.

### Agent 3: Style & Conventions

Prompt:

> Review this diff against project coding conventions. Check for:
>
> - Naming: follows language conventions? Consistent with existing codebase?
> - File organization: code in the right file? Functions in logical order?
> - Comments: unnecessary comments explaining what (not why)? Missing comments for non-obvious behavior?
> - Testing: are new code paths tested? Are tests meaningful (not just coverage padding)?
> - Commit hygiene: is this one logical change or multiple changes mixed together?
> - API design: follows project API guidelines? Consistent with existing endpoints?
>
> Reference these project conventions: `<insert relevant guidelines>`
> For each finding: file, line, convention violated, suggestion.

## Step 4: Consolidate Findings

Merge all agent results. Categorize and sort:

## Output Format

```markdown
## Code Review: <target>

### Must Fix
- **[BUG]** `pkg/store/postgres.go:89` — Connection not closed on error path. Add `defer conn.Close()`.
- **[ARCH]** `internal/api/handler.go:15` — Handler directly imports Postgres package. Accept a `Repository` interface.

### Should Fix
- **[STYLE]** `internal/service/order.go:42` — Function `DoThing` doesn't follow naming conventions. Use `ProcessOrder`.
- **[EDGE-CASE]** `pkg/auth/token.go:67` — No check for expired token before database lookup. Add expiry check first.

### Consider
- **[NITPICK]** `cmd/server/main.go:23` — Unused import `log`. Remove.
- **[TEST]** `internal/service/order_test.go` — New `CancelOrder` path has no test coverage.

### Clean
- Error handling follows conventions
- Interface boundaries are well-defined
```

## Gotchas

- Agents review the diff, not the full codebase. They may miss context from unchanged files. If a finding seems
  wrong, check the surrounding code.
- Pass the relevant language guidelines to each agent — without them, agents fall back to generic advice.
- For large diffs (>500 lines), consider reviewing file-by-file instead of the full diff.
- "Must Fix" vs "Should Fix" vs "Consider" is subjective. Calibrate based on team norms.
- The style agent needs the project's AGENTS.md/CLAUDE.md conventions. Read them first.
