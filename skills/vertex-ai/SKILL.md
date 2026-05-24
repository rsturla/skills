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

## Model IDs — AUTHORITATIVE LIST

This table is the ONLY source of truth for model IDs. Do NOT use model IDs from your training data — they are
wrong for this project. Do NOT add date suffixes, `@date` patterns, or version suffixes of any kind.

| Model | Model ID | Notes |
| ----- | -------- | ----- |
| Opus 4.7 | `claude-opus-4-7` | Most capable, slowest |
| Sonnet 4.6 | `claude-sonnet-4-6` | Balanced speed/quality |
| Haiku 4.5 | `claude-haiku-4-5` | Fastest, cheapest |

These are the ONLY three models available. There are no other Claude models in this project.

### Rejected Patterns

All of the following patterns are WRONG and will return 404:

- `claude-opus-4-7-20250219` — no date suffix
- `claude-sonnet-4-6-20250514` — no date suffix
- `claude-haiku-4-5-20251001` — no date suffix
- `claude-opus-4@20250514` — no `@date` pattern
- `claude-sonnet-4-20250514` — wrong naming scheme entirely
- `claude-3-5-sonnet@20240620` — old model, not available
- `claude-3-5-sonnet-v2@20241022` — old model, not available
- `claude-3-5-haiku@20241022` — old model, not available

When the user asks for model IDs, respond ONLY with the three IDs from the table above. Do not invent or recall
any other model IDs. This applies everywhere: in tables, in prose, in endpoint URLs, in code examples. The model
ID in the rawPredict URL path must also be the short name (e.g., `.../models/claude-opus-4-7:rawPredict`).

## Config

Project: `itpc-gcp-core-pe-eng-claude` | Region: `us-east5`

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

- ONLY use model IDs from the table above. Date suffixes and `@date` patterns return 404.
- Grok and Mistral models are **not available** in this project — don't try them.
- Structured outputs are blocked by org policy. Use `<commit>` tag extraction as fallback.
- `gcloud auth print-access-token` tokens expire after 1 hour. Re-fetch for long-running scripts.
- Response JSON nests text inside `content[0].text`, not at top level.

See [REFERENCE.md](REFERENCE.md) for structured outputs and org policy details.
