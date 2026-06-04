# /api-contract

Run full API contract validation: find the schema, scan frontend API calls, compare endpoints, check request/response shapes, and report all mismatches.

## Usage

```
/api-contract
```

No arguments. Runs all three checks in sequence:
1. Schema discovery and parsing
2. Frontend API call scanning
3. Endpoint matching and field validation

## What it does

Invokes the `contract-validate` skill to produce a full validation report covering:
- Matched endpoints (frontend call exists in schema)
- Unmatched calls (frontend calls an endpoint not in schema)
- Method mismatches (wrong HTTP verb)
- Missing required request fields
- Missing response fields the frontend tries to read
- Unused schema endpoints (defined but never called)

## Output

Prints a structured report to the terminal. No files are modified.

## Example output

```
API Contract Validation Report
================================
Schema: openapi.yaml (OpenAPI 3.0.3)
Frontend files scanned: 24
Endpoints found in frontend: 11

MATCHED (8): GET /users, GET /users/{id}, POST /users ...
UNMATCHED (2): POST /users/{id}/avatar, DELETE /sessions
FIELD MISMATCHES (1): POST /users missing required field "role"

Summary: 3 issues require attention
```

## Skill

Executes: `skills/contract-validate/SKILL.md`
