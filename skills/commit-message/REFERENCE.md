# Commit Message — Reference

## Vertex AI Call Pattern

```bash
PROJECT="${CLAUDE_COMMIT_PROJECT:-itpc-gcp-core-pe-eng-claude}"
LOCATION="${CLAUDE_COMMIT_LOCATION:-global}"
MODEL="${CLAUDE_COMMIT_MODEL:-claude-haiku-4-5}"
TOKEN=$(gcloud auth print-access-token)

curl -s -X POST \
  "https://aiplatform.googleapis.com/v1/projects/${PROJECT}/locations/${LOCATION}/publishers/anthropic/models/${MODEL}:rawPredict" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json; charset=utf-8" \
  -d "$(jq -n --arg prompt "$PROMPT" '{
    anthropic_version: "vertex-2023-10-16",
    messages: [{"role": "user", "content": $prompt}],
    max_tokens: 1024,
    stream: false
  }')"
```

## Structured Output Schema

Attempt structured outputs first (`output_config.format` with `json_schema`), fall back to `<commit>` tags if org
policy blocks it.

```json
{
  "type": "object",
  "properties": {
    "subject": {"type": "string"},
    "bullets": {"type": "array", "items": {"type": "string"}}
  },
  "required": ["subject", "bullets"],
  "additionalProperties": false
}
```

### Tag Fallback

Add to prompt when structured outputs unavailable:

```text
Wrap your entire commit message in <commit> </commit> tags.
Output NOTHING outside the tags.
```

## Git Hook

This skill also ships as a standalone `prepare-commit-msg` git hook for automatic commit message improvement.

```bash
# Global (all repos)
git config --global core.hooksPath ~/.local/share/git-hooks

# Per-repo
ln -s ~/.local/share/git-hooks/prepare-commit-msg /path/to/repo/.git/hooks/

# Skip for one commit
SKIP_CLAUDE_COMMIT=1 git commit -m "message"
```
