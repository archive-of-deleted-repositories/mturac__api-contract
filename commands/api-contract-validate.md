# /api-contract-validate

Validate that frontend API calls match the backend OpenAPI/Swagger schema. Detects mismatched endpoints, wrong HTTP methods, missing required fields, and response field discrepancies.

## Usage

```
/api-contract-validate
/api-contract-validate --schema path/to/openapi.yaml
/api-contract-validate --src path/to/frontend/src
```

## Options

- `--schema <path>` — override schema file path (default: auto-discover openapi.yaml / swagger.json)
- `--src <path>` — override frontend source directory (default: `src/`)
- `--strict` — treat unused schema endpoints as errors (non-zero exit)
- `--json` — output report as JSON instead of human-readable text

## What it checks

1. Every endpoint called by the frontend exists in the schema with the correct HTTP method
2. Request body fields match schema requirements (no missing required fields)
3. Response fields read by the frontend exist in the schema response shape

## Skill

Executes: `skills/contract-validate/SKILL.md`
