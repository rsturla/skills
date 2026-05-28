---
name: code-review
description: >
  Strict multi-agent code review for maintainability, correctness, and project conventions. Pushes code-judo
  simplification, boundary cleanliness, and anti-spaghetti structure — not cosmetic nits. Use when the user asks
  for a code review, maintainability audit, or feedback on a diff or PR — even if they just say "what do you think?"
  or "review this".
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
  version: 2.0.0
---

# Code Review

Multi-agent review: gather diff and context once, run three parallel specialists, merge into one prioritized report.
Behavior-correct code that makes the codebase messier is **not** enough to approve.

## Process

1. Detect review target and collect diff + changed file paths
2. Read full contents of changed files and load language guidelines (Step 2)
3. Launch three agents in parallel with shared context (Step 3)
4. Merge, dedupe, and output in priority order (Step 4)

## Step 1: Detect Review Target

```bash
# Staged changes first
if [ -n "$(git diff --cached --name-only)" ]; then
    DIFF_CMD="git diff --cached"
    TARGET="staged changes"
elif gh pr view --json number &>/dev/null 2>&1; then
    DIFF_CMD="gh pr diff"
    TARGET="PR #$(gh pr view --json number -q .number)"
elif glab mr view &>/dev/null 2>&1; then
    DIFF_CMD="glab mr diff"
    TARGET="MR !$(glab mr view --output json 2>/dev/null | jq -r .iid)"
else
    BASE=$(git merge-base HEAD origin/main 2>/dev/null || echo "origin/main")
    DIFF_CMD="git diff $BASE...HEAD"
    TARGET="branch $(git branch --show-current) vs main"
fi

DIFF=$($DIFF_CMD)
CHANGED_FILES=$($DIFF_CMD --name-only)
```

For diffs over ~500 lines, review file-by-file instead of one giant prompt.

## Step 2: Context

Read each changed file in full (agents see the diff, not always enough surrounding code).

Detect language from extensions and load guidelines from AGENTS.md Language Guides:

| Extension / path | Guidelines |
| --- | --- |
| `.go` | Go Coding Guidelines |
| `.rs` | Rust Coding Guidelines |
| `.py` | Python Coding Guidelines |
| `.tf`, `.hcl` | OpenTofu + Terragrunt Guidelines |
| `Containerfile` | Containerfile Guidelines |
| `.github/workflows/*` | GitHub Actions Guidelines |

Pass guideline excerpts into every agent prompt.

## Rubric

Apply to everything the diff touches. Trace cross-file impact at module boundaries.

**Ambition** — search for code-judo: same behavior, fewer branches, layers, or concepts. Prefer deleting complexity over
moving it. Do not stop at "slightly cleaner."

**Structure** — file size, layering, and branching:

- No file crossing ~1000 lines without strong justification; decompose first when the PR pushes a file over the line.
- No ad-hoc conditionals bolted onto busy shared paths; use a dedicated abstraction, policy, or module.
- Logic in the canonical layer; reuse existing helpers instead of near-duplicates.
- Abstractions must earn their keep — no thin wrappers, identity types, or pass-through indirection.
- Prefer explicit types and contracts over `any`, heavy casts, or optionality that hides invariants.

**Correctness** — nil/null, concurrency, resource leaks, off-by-one, missing error paths, incomplete refactors.

**Conventions** — naming, file placement, meaningful tests, API shape per project docs.

**Orchestration** — flag unnecessary serialization of independent work or non-atomic partial updates when a simpler
structure is obvious (not micro-optimization nits).

When `security-review` runs in parallel, skip secrets, auth, and injection — this skill owns design and
maintainability only.

## Step 3: Launch Agents

Use the Agent tool. **One message, three parallel calls.** Each prompt must include:

- `### Git / diff output` — full `DIFF`
- `### Changed files` — list from `CHANGED_FILES`
- `### File contents` — full text of changed files you read
- `### Guidelines` — relevant convention excerpts
- Instruction to apply only the **Rubric** sections named for that agent

### Agent 1: Structure and maintainability

Rubric sections: Ambition, Structure, Orchestration.

For each finding: file, line, issue, concrete restructuring suggestion. High conviction; skip cosmetic nits when
structural problems exist.

### Agent 2: Correctness and edge cases

Rubric section: Correctness.

For each finding: file, line, description, severity (`BUG` / `RISK` / `NITPICK`), fix.

### Agent 3: Conventions and style

Rubric sections: Conventions (+ Structure only where it overlaps naming, placement, or API shape).

For each finding: file, line, convention violated, suggestion.

## Step 4: Consolidate

Merge agent output. Deduplicate. Sort findings into the sections below — **do not** bury structural issues under
style nits.

### Approval bar

Block approval when any of these are visible and unjustified:

- Plausible code-judo simplification was skipped; incidental complexity preserved
- File grew past ~1000 lines without decomposition
- Feature logic scattered across shared paths as special-case branches
- New wrapper, cast-heavy contract, or wrong-layer logic with a clear canonical home elsewhere

"Works" alone is insufficient.

## Output Format

```markdown
## Code Review: <target>

### Must Fix
- **[STRUCT]** `pkg/foo.go:120` — Special-case branch in shared handler. Move behind `OrderPolicy`.
- **[BUG]** `pkg/store/postgres.go:89` — Connection not closed on error path. `defer conn.Close()`.

### Should Fix
- **[MAINTAIN]** `internal/service/order.go:42` — 1.2k-line file after PR. Extract `billing` module first.
- **[EDGE]** `pkg/auth/token.go:67` — Expiry not checked before DB lookup.

### Consider
- **[STYLE]** `cmd/server/main.go:23` — Unused import. Remove.
- **[TEST]** `internal/service/order_test.go` — `CancelOrder` path untested.

### Clean
- No obvious structural regression in touched modules
```

Priority when merging (highest first): structural regression → missed simplification → spaghetti / branching growth →
boundary and type-contract problems → file size / decomposition → modularity → legibility.

## Gotchas

- Agents only see what you pass. Wrong finding → read surrounding unchanged code before reporting.
- Large diffs: split by file or package.
- Performance (N+1, O(n²)) belongs in Agent 2 unless it is clearly a structural orchestration smell.
- Calibrate Must / Should / Consider to team norms; default strict on structure, pragmatic on nits.
