---
name: vertex-ai
description: >
  Query and call Claude models on Vertex AI. Model reference, endpoint patterns, and quick API calls. Use when
  the user asks about Vertex AI, model availability, wants to call Claude via GCP, or mentions rawPredict, Vertex
  endpoints, or GCP AI — even if they just say "which Claude model", "newest model", "latest model", "call Claude
  from Go/Rust/Python", or ask about model IDs without mentioning Vertex AI.
compatibility: Requires gcloud CLI (authenticated), curl, and jq.
allowed-tools:
  - Bash(gcloud auth print-access-token)
  - Bash(curl -s -X POST https://*-aiplatform.googleapis.com/*)
  - Bash(jq *)
metadata:
  author: rsturla
  version: 1.0.0
---

# Vertex AI Claude Helper

## Config

Project: `itpc-gcp-core-pe-eng-claude` | Region: `us-east5`

Use **short model names** without date suffix (e.g. `claude-haiku-4-5`, not `claude-haiku-4-5-20251001`).

## Endpoint

```text
POST https://${REGION}-aiplatform.googleapis.com/v1/projects/${PROJECT}/locations/${REGION}/publishers/anthropic/models/${MODEL}:rawPredict
Authorization: Bearer $(gcloud auth print-access-token)
```

Request body requires `"anthropic_version": "vertex-2023-10-16"`.

## Quick Call

```bash
TOKEN=$(gcloud auth print-access-token) && \
curl -s -X POST \
  "https://us-east5-aiplatform.googleapis.com/v1/projects/itpc-gcp-core-pe-eng-claude/locations/us-east5/publishers/anthropic/models/${MODEL}:rawPredict" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json; charset=utf-8" \
  -d "$(jq -n --arg prompt "$PROMPT" '{
    anthropic_version: "vertex-2023-10-16",
    messages: [{"role": "user", "content": $prompt}],
    max_tokens: 1024,
    stream: false
  }')"
```

Parse response: `jq -r '.content[0].text'`

## Test Model Availability

```bash
TOKEN=$(gcloud auth print-access-token) && \
curl -s -X POST \
  "https://us-east5-aiplatform.googleapis.com/v1/projects/itpc-gcp-core-pe-eng-claude/locations/us-east5/publishers/anthropic/models/${MODEL}:rawPredict" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"anthropic_version":"vertex-2023-10-16","messages":[{"role":"user","content":"hi"}],"max_tokens":8}' \
  | jq -r 'if .error then "NO: " + .error.message else "YES" end'
```

## Gotchas

- Use **short model names** (`claude-haiku-4-5`), not dated versions (`claude-haiku-4-5-20251001`). Dated names
  return 404.
- Grok and Mistral models are **not available** in this project — don't try them.
- Structured outputs are blocked by org policy. Use `<commit>` tag extraction as fallback.
- `gcloud auth print-access-token` tokens expire after 1 hour. Re-fetch for long-running scripts.
- Response JSON nests text inside `content[0].text`, not at top level.

See [REFERENCE.md](REFERENCE.md) for structured outputs and org policy details.
