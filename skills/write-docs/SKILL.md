---
name: write-docs
description: >
  Write clear, structured technical documentation. README files, usage guides, API docs, and contributor docs with
  consistent voice and formatting. Use when the user needs to write or improve documentation — even if they just say
  "add a README" or "document this."
allowed-tools:
  - Read
  - Write
metadata:
  author: rsturla
  version: 1.0.0
---

# Documentation Writing Guide

## Process

1. Read existing docs to understand current state
1. Outline sections following what→how→reference principle
1. Draft content following voice and formatting rules
1. Review against gotchas checklist

## Voice and Tone

- Professional, direct, third person — no "we", no "I", no "you should"
- Imperative mood for instructions: "Start the service", "Create a file"
- Present tense: "This tool provides..." not "This tool will provide..."
- First paragraph: one sentence explaining what the thing does and why it matters
- No marketing language, no superlatives, no filler

## Structure Principles

- **What → How → Reference** — reader understands what it is before being told how to use it
- Lead with the simplest example, then layer complexity
- Group related content under clear headings — don't make readers hunt
- Keep pages focused on one topic. Split long docs rather than scrolling forever.

## Formatting

- **Headings**: title case for H1, sentence case for H2+
- **Numbered lists**: `1.` for all items (auto-numbered), 4-space indent for nested content
- **Placeholders**: `<angle_brackets_with_underscores>` — e.g. `<file_path>`, `<port>`
- **Code blocks**: always include language tag (`bash`, `yaml`, `json`, `go`, `hcl`, `text`)
- **Commands**: long flags in docs (`--output` not `-o`) for clarity
- **Inline code**: backtick file paths, commands, package names, config keys
- **Links**: descriptive text, never "click here" or bare URLs
- **Bold**: first introduction of key terms only. Not for entire sentences.
- **Paragraphs**: 2-3 sentences max. Break up walls of text.

## Code Examples

- Every capability described needs a runnable example
- Show full commands, don't abbreviate
- Use `<placeholder>` for user-supplied values
- Multi-step procedures use numbered lists with indented code blocks:

```markdown
1. Create the configuration:

    ```yaml
    server:
      port: 8080
    ```

1. Start the service:

    ```bash
    ./server --config <config_path>
    ```
```

## Gotchas

- Numbered lists: `1.` for every item — markdown auto-numbers. Makes reordering easy.
- Indent code under numbered lists by 4 spaces, not 2.
- Placeholders are `<snake_case>`, not `${CAPS}` or `[brackets]`.
- Don't mix short and long flags in the same doc.
- Avoid "Note:" and "Important:" callouts unless genuinely critical — overuse trains readers to ignore them.
