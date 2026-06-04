# /api-contract-diff

Compare two versions of an API schema to find breaking changes. Classify each change as breaking (removes or modifies existing contract) or additive (adds new capabilities).

## Usage

```
/api-contract-diff
/api-contract-diff main HEAD
/api-contract-diff v1.0.0 v1.1.0
/api-contract-diff openapi-v1.yaml openapi-v2.yaml
/api-contract-diff main feature/new-endpoints
```

## Arguments

- `[ref1]` — first version: a git ref (branch, tag, commit SHA) or a file path. Default: `main`
- `[ref2]` — second version: a git ref or file path. Default: current working tree

If both arguments are git refs, the skill runs `git show <ref>:<schema-file>` to extract each version.
If both are file paths, it reads both files directly.

## Breaking change categories detected

| Category | Severity |
|---|---|
| Removed endpoint | CRITICAL |
| Field added to required[] | HIGH |
| Response field removed | HIGH |
| Field type changed | HIGH |
| Path parameter renamed/changed | HIGH |
| Auth requirement added | HIGH |
| Enum values narrowed | MEDIUM |
| Success response code changed | MEDIUM |

## Additive changes (safe)

New endpoints, new optional fields, new response fields, new enum values, required → optional.

## Skill

Executes: `skills/contract-diff/SKILL.md`
