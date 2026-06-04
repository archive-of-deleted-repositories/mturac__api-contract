---
name: contract-diff
description: Compare two versions of an API spec to detect breaking changes before deploy
tags:
  - api
  - openapi
  - breaking-changes
  - schema-diff
trigger: When the user asks to diff API versions, find breaking changes, or compare schemas between branches or commits
---

# Contract Diff

Compare two versions of an OpenAPI/Swagger spec to identify breaking changes vs additive changes. Useful before deploying a backend to verify no frontend-breaking changes slip through.

## Steps

### 1. Accept Inputs

Accept two spec sources. They can be provided as:

- **Two file paths**: `openapi-v1.yaml openapi-v2.yaml`
- **Two git refs**: `main HEAD`, `v1.0.0 v1.1.0`, `main feature/new-api`
- **One file path + one git ref**: compare current file against `main` branch version
- **Default (no args)**: compare `main` branch version of the schema file against the working tree version

If using git refs:
1. Find the schema file (same priority order as contract-validate step 1).
2. Run `git show <ref>:<schema-path>` to extract each version.
3. Parse both outputs as YAML or JSON.

If the file doesn't exist at a given ref, report that and stop.

### 2. Parse Both Specs

Parse both spec versions into structured objects. Extract from each:

- **Paths map**: `{ "GET /users": { parameters, requestBody, responses }, ... }`
- **Schemas map**: all `$ref` definitions from `components/schemas` or `definitions`
- **Security schemes**: OAuth, API key, JWT definitions

Resolve all `$ref` references before comparing so the diff operates on concrete shapes, not pointers.

### 3. Detect Breaking Changes

A change is **breaking** if it can cause an existing client to fail or return incorrect data without modification.

Scan for these breaking change categories:

#### 3a. Removed Endpoints
- Path+method exists in v1 but not v2.
- Severity: **CRITICAL** — any client calling this will get 404.

#### 3b. Removed Required Request Fields
- A field listed in v1 `required[]` is absent from v2 `required[]` — this is **additive** (safe).
- A field NOT in v1 `required[]` is added to v2 `required[]` — this is **breaking** (existing clients don't send it).
- Severity: **HIGH**

#### 3c. Removed Response Fields
- A field present in v1 response schema is absent from v2 response schema.
- Severity: **HIGH** — clients reading this field will get `undefined`.

#### 3d. Changed Field Types
- A field's `type` or `format` changed between versions (e.g., `string` → `integer`, `date` → `date-time`).
- Severity: **HIGH**

#### 3e. Changed Response Status Codes
- A success response (2xx) changed to a different 2xx code (e.g., 200 → 201).
- An error response code changed (e.g., 404 → 410).
- Severity: **MEDIUM**

#### 3f. Changed Path Parameters
- A path parameter was renamed (e.g., `/users/{id}` → `/users/{userId}`).
- A path parameter type changed.
- Severity: **HIGH**

#### 3g. Changed Authentication Requirements
- An endpoint that previously required no auth now requires auth.
- Auth scheme changed.
- Severity: **HIGH**

#### 3h. Narrowed Enum Values
- An enum field had values removed (fewer valid values means existing clients may send now-invalid values).
- Severity: **MEDIUM**

### 4. Detect Additive (Safe) Changes

A change is **additive** if existing clients continue to work unchanged.

- New endpoint added (new path+method)
- New optional request field added
- New response field added
- New enum value added
- Required field made optional
- New response status code added alongside existing ones

Report these separately as safe changes.

### 5. Output Breaking Changes Table

```
API Contract Diff: main → HEAD
Schema file: openapi.yaml
================================

BREAKING CHANGES (4)
┌─────────┬──────────────────────────────┬────────────────────────────────────────────────────────┐
│ Sev     │ Location                     │ Change                                                 │
├─────────┼──────────────────────────────┼────────────────────────────────────────────────────────┤
│ CRITICAL│ DELETE /users/{id}           │ Endpoint removed                                       │
│ HIGH    │ POST /users → body.role      │ Field added to required[] — existing clients won't send │
│ HIGH    │ GET /users/{id} → res.email  │ Field removed from 200 response schema                 │
│ MEDIUM  │ POST /auth/login             │ Success response changed: 200 → 201                    │
└─────────┴──────────────────────────────┴────────────────────────────────────────────────────────┘

ADDITIVE CHANGES (3) — safe for existing clients
  + GET /users/{id}/permissions    New endpoint
  + POST /users body.avatarUrl     New optional field
  + GET /users response.createdAt  New response field

Summary: 4 breaking changes detected. Review before deploying.
```

**Guardrail**: Read-only. Does not modify files or commit anything.
