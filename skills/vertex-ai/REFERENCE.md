# Vertex AI — Reference

## Structured Outputs

Uses `output_config.format` with `type: "json_schema"`. May require org policy change —
`structured_outputs` must be added to `constraints/vertexai.allowedPartnerModelFeatures`.

```json
{
  "output_config": {
    "format": {
      "type": "json_schema",
      "schema": {
        "type": "object",
        "properties": {"key": {"type": "string"}},
        "required": ["key"],
        "additionalProperties": false
      }
    }
  }
}
```

## Org Policy Constraints

| Constraint | Controls |
| ---------- | -------- |
| `constraints/vertexai.allowedModels` | Which models are accessible |
| `constraints/vertexai.allowedPartnerModelFeatures` | Features like `structured_outputs`, `web_search` |

To request access to a blocked model or feature, ask the org admin to add the appropriate value
(e.g. `publishers/anthropic/models/claude-sonnet-4-6:structured_outputs`).
