---
name: write-docs
description: >
  Write clear, structured technical documentation. README files, usage guides, and contributor docs with consistent
  voice, formatting, and section ordering. Use when the user needs to write or improve documentation — even if they
  just say "add a README" or "document this."
metadata:
  author: rsturla
  version: 1.0.0
---

# Documentation Writing Guide

Write clear, structured technical documentation following these conventions.

## Voice and Tone

- **Professional, direct, third person** — no "we", no "I", no "you should"
- Use imperative mood for instructions: "Start the service", "Create a volume"
- Present tense: "This tool provides..." not "This tool will provide..."
- Explain what something *is* before explaining how to *use* it
- First paragraph: one-sentence description of what the thing does and why it matters
- No marketing language, no superlatives, no unnecessary adjectives

## Section Ordering

Follow this structure, adapting sections to the subject:

```markdown
# <Project/Tool Name>

<One paragraph: what it is, what it does, why it's useful.>

## Prerequisites

## Quick Start

## Usage

### Basic Usage
### <Feature-specific sections>

## Configuration

## Troubleshooting

## Contributing
```

Principle: **what → how → reference**. Reader should understand what it is before being told how to use it.

## Formatting Rules

- **Headings**: title case for H1, sentence case for H2+
- **Numbered lists**: use `1.` for all items (auto-numbered), 4-space indent for nested content
- **Placeholders**: use `<angle_brackets_with_underscores>` — e.g. `<container_name>`, `<config_path>`
- **Code blocks**: always include language tag (`bash`, `yaml`, `json`, `go`, `hcl`, `text`)
- **Commands**: use long flags (`--detach` not `-d`, `--volume` not `-v`) in documentation
- **Inline code**: backtick file paths, commands, package names, ports, config keys
- **Links**: descriptive text, not "click here" or bare URLs
- **Bold**: key terms on first introduction and emphasis only. Not for entire sentences.

## Code Examples

- Every usage section needs a runnable example
- Show the full command, don't abbreviate
- Use `<placeholder>` syntax for user-supplied values
- Group related steps in numbered lists with code blocks indented under each step:

```markdown
1. Create the configuration file:

    ```yaml
    server:
      port: 8080
      host: 0.0.0.0
    ```

1. Start the service:

    ```bash
    ./server --config <path_to_config>
    ```
```

## README Structure

For project READMEs, include at minimum:

1. **Title + one-line description** — what it is
2. **Quick start** — fastest path to running it
3. **Usage** — detailed examples for each feature
4. **Configuration** — environment variables, config files, flags
5. **Contributing** — how to build, test, submit changes

## Gotchas

- Numbered lists use `1.` for every item — markdown auto-numbers. Makes reordering easy.
- Indent code blocks under numbered lists by 4 spaces, not 2.
- Placeholders are `<snake_case>`, not `${CAPS}` or `[brackets]`.
- Don't mix short and long flags in the same doc — pick long flags for clarity.
- Avoid "Note:" and "Important:" callouts unless genuinely critical. Overuse trains readers to ignore them.
- Keep paragraphs short — 2-3 sentences max. Wall of text loses readers.
