---
name: security-review
description: >
  Security-focused code review using multiple specialized agents. Checks for injection, auth issues, secrets,
  supply chain risks, and OWASP patterns. Use when the user asks for a security review, mentions vulnerabilities
  in their code, or says "is this secure?" — even if they just say "check for security issues."
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

# Security Review

Multi-agent security review that produces actionable findings.

## Process

1. Detect review target (staged changes, branch diff, or PR)
2. Launch 3 specialized agents in parallel
3. Consolidate findings into a single report
4. Prioritize by severity and present action items

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

Get the diff and changed files:

```bash
DIFF=$($DIFF_CMD)
CHANGED_FILES=$($DIFF_CMD --name-only)
```

## Step 2: Launch Specialized Agents

Launch **3 agents in parallel** using the Agent tool. Include the full diff output and changed file list in each
agent's prompt. For CRITICAL/HIGH findings, instruct agents to Read the surrounding file context before finalizing.

### Agent 1: Injection & Input Validation

Prompt:

> Review this diff for injection vulnerabilities and input validation issues. Check for:
>
> - SQL injection (string concatenation in queries, missing parameterization)
> - Command injection (user input in shell commands, `exec`, `subprocess`, `os/exec`)
> - XSS (unescaped output in HTML templates, `innerHTML`, `dangerouslySetInnerHTML`)
> - Path traversal (user input in file paths without sanitization)
> - SSRF (user-controlled URLs in HTTP requests)
> - Deserialization of untrusted data (`pickle`, `yaml.load`, `json.Unmarshal` into `interface{}`)
>
> For each finding: file, line, vulnerability type, severity (CRITICAL/HIGH/MEDIUM), and fix.

### Agent 2: Auth, Secrets & Access Control

Prompt:

> Review this diff for authentication, authorization, and secrets issues. Check for:
>
> - Hardcoded credentials, API keys, tokens, passwords in source code
> - Missing authentication checks on endpoints/handlers
> - Broken access control (missing authorization after authentication)
> - IDOR (user A accessing user B's resources via predictable IDs without ownership check)
> - Insecure session handling (predictable tokens, missing expiry)
> - Secrets in logs (logging sensitive fields, request/response bodies with credentials)
> - Weak cryptography (MD5, SHA1 for passwords, ECB mode, hardcoded IVs)
> - Missing TLS enforcement
>
> For each finding: file, line, issue type, severity, and fix.

### Agent 3: Dependency & Supply Chain

Prompt:

> Review this diff for dependency and supply chain security issues. Check for:
>
> - New dependencies added: are they well-known and maintained?
> - Version changes: any downgrades or jumps to suspiciously new major versions?
> - Typosquatting: package names that look like misspellings of popular packages
> - Removed security-related dependencies (auth libraries, crypto, sanitizers)
> - Unsafe dependency patterns (pinning to `latest`, missing lockfile updates)
> - `go.sum`/`Cargo.lock`/`requirements.txt` changes without corresponding code changes
>
> For each finding: file, line, package name, concern, severity, and recommendation.

## Step 3: Consolidate Findings

Merge all agent results into a single report, deduplicate, and sort by severity.

## Output Format

```markdown
## Security Review: <target>

### Critical
- **[INJECTION]** `src/api/handler.go:42` — SQL concatenation with user input. Use parameterized query.

### High
- **[SECRETS]** `config/app.yaml:15` — API key hardcoded. Move to environment variable or secret manager.
- **[AUTH]** `src/middleware/auth.go:28` — Missing authorization check. Endpoint accessible to any authenticated user.

### Medium
- **[SUPPLY-CHAIN]** `go.sum` — New dependency `github.com/foo/bar` added. Low download count, verify legitimacy.

### Clean
- No injection issues found in template rendering
- TLS enforcement present in server configuration
```

## Gotchas

- Agents review the diff, not the full codebase. A missing auth check in existing code won't be caught unless the
  diff touches that file.
- False positives are expected. Each finding should be verified before acting.
- `gh pr diff` / `glab mr diff` require authenticated CLI and a PR/MR context. Falls back to branch diff if unavailable.
- Supply chain agent is most useful when `go.sum`, `Cargo.lock`, or `requirements.txt` are in the diff.
- If the `dependency-audit` skill is available, delegate deep supply chain analysis to it and keep Agent 3 lightweight.
- If running alongside `code-quality-review`, this skill owns security findings. Let code-quality-review focus on
  maintainability and structure.
