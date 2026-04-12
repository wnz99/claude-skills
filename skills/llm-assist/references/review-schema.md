# Structured Review Output Schema

When running review mode, you can optionally request structured JSON output.

> **Note:** The `--output-schema` flag is only available with the Codex
> provider. For OpenCode, include "Respond with JSON matching this schema:"
> in the prompt and append the schema below.

## Schema

```json
{
  "type": "object",
  "properties": {
    "findings": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "severity": {
            "type": "string",
            "enum": ["P0", "P1", "P2", "P3"]
          },
          "file": {
            "type": "string"
          },
          "line_range": {
            "type": "string"
          },
          "title": {
            "type": "string"
          },
          "description": {
            "type": "string"
          },
          "suggestion": {
            "type": "string"
          }
        },
        "required": ["severity", "file", "title", "description"],
        "additionalProperties": false
      }
    },
    "summary": {
      "type": "string"
    },
    "verdict": {
      "type": "string",
      "enum": ["APPROVE", "APPROVE_WITH_CONCERNS", "REQUEST_CHANGES"]
    }
  },
  "required": ["findings", "summary", "verdict"],
  "additionalProperties": false
}
```

## Usage

### Codex

```bash
SCHEMA_FILE=$(mktemp /tmp/codex-schema-XXXXXX.json)
cat > "$SCHEMA_FILE" << 'SCHEMA'
[paste schema above]
SCHEMA

codex exec \
  -s read-only \
  --ephemeral \
  --output-schema "$SCHEMA_FILE" \
  -o "$OUTPUT_FILE" \
  - < "$PROMPT_FILE"
```

### OpenCode

Append the schema to the prompt file before the `## Instructions` section:

```markdown
## Output Format
Respond ONLY with valid JSON matching this schema:
[paste schema above]
```

Then run: `opencode run "Follow the instructions in the attached file" -f "$PROMPT_FILE" > "$OUTPUT_FILE" 2>&1`

## Parsing

The output file will contain JSON matching the schema. Parse it to
extract findings by severity and compare with your own review:

```
findings[].severity → P0/P1/P2/P3
findings[].file     → file path
findings[].title    → one-line summary
findings[].description → detailed explanation
findings[].suggestion  → suggested fix (optional)
verdict             → overall assessment
summary             → narrative summary
```
