---
name: commit-message
description: Generate or improve git commit messages from staged changes using Claude on Vertex AI. Conventional Commits format, validates line lengths, retries on failure. Use when the user wants to write, generate, or improve a commit message, asks for help with git commits, or invokes /commit-message — even if they just say "commit this" or "what should the commit say."
compatibility: Requires gcloud CLI (authenticated), curl, and jq.
allowed-tools:
  - Bash(git diff --cached*)
  - Bash(git log*)
  - Bash(gcloud auth print-access-token)
  - Bash(curl -s -X POST https://*-aiplatform.googleapis.com/*)
  - Bash(jq *)
metadata:
  author: rsturla
  version: 1.0.0
---

# Commit Message Generator

Generate or improve commit messages from staged changes using Claude on Vertex AI.

## Process

1. Read staged diff: `git diff --cached --stat` + `git diff --cached` (truncate to 8000 chars)
2. If user provided a draft, improve it. Otherwise generate from scratch.
3. Call Claude on Vertex AI
4. Validate result, retry with error feedback if invalid
5. Present commit message to user

## Example Output

```text
feat: add database connection pool and migrations

- Add SQLite pool with WAL mode, auto-commit/rollback, and connection limits
- Add schema migrations for users and sessions tables
- Import DatabasePool in app entry point
```

## Commit Message Rules

- **Format**: Conventional Commits (`feat`/`fix`/`refactor`/`docs`/`test`/`chore`/`ci`/`build`/`perf`/`style`)
- **Subject**: imperative mood, lowercase after prefix, max 50 chars, no trailing period
- **Body**: bullet points for non-trivial changes, each line max 72 chars
- **Trivial changes**: subject only, no body

## Validation

1. Subject ≤ 50 chars
2. Starts with Conventional Commits prefix
3. No trailing period
4. Line 2 blank (if body exists)
5. Body lines ≤ 72 chars

Retry with error feedback on failure.

## Configuration

| Env Var | Default | Description |
| ------- | ------- | ----------- |
| `CLAUDE_COMMIT_PROJECT` | `itpc-gcp-core-pe-eng-claude` | GCP project |
| `CLAUDE_COMMIT_REGION` | `us-east5` | GCP region |
| `CLAUDE_COMMIT_MODEL` | `claude-haiku-4-5` | Model to use |
| `CLAUDE_COMMIT_RETRIES` | `3` | Max retry attempts |

## Gotchas

- Truncate diff to 8000 chars — large diffs blow token limits and produce worse messages
- Structured outputs blocked by org policy — use `<commit>` tag extraction as fallback
- Subject ≤50 is strict. Models frequently produce 51-55 char subjects — validation + retry catches this
- Merge/squash/amend commits should be skipped — don't rewrite those messages

See [REFERENCE.md](REFERENCE.md) for Vertex AI call patterns, structured output schema, and git hook installation.
