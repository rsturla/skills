# Write a Skill — Reference

## Frontmatter Fields

| Field | Required | Description |
| ----- | -------- | ----------- |
| `name` | Yes | Unique kebab-case identifier |
| `description` | Yes | Max 1024 chars. First sentence: what it does. Second: "Use when [triggers]" |
| `allowed-tools` | No | Whitelist of tool patterns the skill needs |
| `metadata.author` | No | Author identifier |
| `metadata.version` | No | Semver version |

## When to Add Scripts

Add utility scripts when:

- Operation is deterministic (validation, formatting)
- Same code would be generated repeatedly
- Errors need explicit handling

Scripts save tokens and improve reliability vs generated code.

## When to Split Files

Split into separate files when:

- SKILL.md exceeds 100 lines
- Content has distinct domains
- Advanced features are rarely needed

## Description Examples

**Good**:

```text
Generate or improve git commit messages from staged
changes using Claude on Vertex AI. Conventional
Commits format, validates line lengths, retries on
failure. Use when user says "write a commit message",
"improve this commit", or invokes /commit-message.
```

**Bad**:

```text
Helps with commits.
```

## Installation

Skills are installed via `bunx skills`:

```bash
bunx skills add rsturla/skills -g        # global
bunx skills add rsturla/skills --list     # preview
```
