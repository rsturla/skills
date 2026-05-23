---
name: write-a-skill
description: Create new agent skills following the agentskills.io spec with proper structure, progressive disclosure, and bundled resources. Use when the user wants to create, write, or build a new skill — even if they just say "automate this workflow" or "teach the agent to do X."
metadata:
  author: rsturla
  version: 1.0.0
---

# Writing Skills

Based on the [agentskills.io specification](https://agentskills.io/specification).

## Process

1. **Start from real expertise** — extract from a hands-on task or existing docs, not generic LLM knowledge
2. **Draft the skill** — SKILL.md under 500 lines, split to REFERENCE.md if needed
3. **Review with user** — iterate, add gotchas from real failures
4. **Test triggering** — verify the description activates on relevant prompts

## Skill Structure

```text
skill-name/
├── SKILL.md           # Required: metadata + instructions
├── references/        # Optional: detailed docs loaded on demand
├── scripts/           # Optional: deterministic operations
└── assets/            # Optional: templates, schemas
```

- `name` must match parent directory, kebab-case, max 64 chars
- Follow [AGENTS.md](../../AGENTS.md)

## SKILL.md Template

```markdown
---
name: skill-name
description: What it does. Use when [triggers] — even if they don't explicitly mention [domain].
compatibility: Requires [tools/env].
allowed-tools:
  - Bash(relevant commands*)
metadata:
  author: rsturla
  version: 1.0.0
---

# Skill Name

## Quick Start
[Minimal working example]

## Workflows
[Step-by-step for complex tasks]

## Gotchas
[Non-obvious facts that prevent common mistakes]

## Advanced
[See [REFERENCE.md](references/REFERENCE.md)]
```

## Description

The description is **the only thing your agent sees** when deciding to load.

- Max 1024 chars
- First sentence: what it does
- Second: "Use when [triggers] — even if they don't explicitly mention [X]"
- Be pushy — list contexts where the skill applies
- Focus on user intent, not implementation

## Key Patterns

- **Gotchas sections** — highest-value content; concrete corrections to mistakes the agent will make
- **Defaults, not menus** — pick one approach, mention alternatives briefly
- **Validation loops** — have the agent check its own work before proceeding
- **Progressive disclosure** — SKILL.md has core instructions, references/ has details loaded on demand

## Review Checklist

- [ ] Description includes triggers with "even if" broadening
- [ ] `compatibility` field set if env requirements exist
- [ ] SKILL.md under 500 lines (~5000 tokens)
- [ ] Gotchas section for non-obvious facts
- [ ] No time-sensitive info that will rot
- [ ] Concrete examples included
- [ ] `name` matches directory name

See [REFERENCE.md](REFERENCE.md) for frontmatter field details and script guidelines.
