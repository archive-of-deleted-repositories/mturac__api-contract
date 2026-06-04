---
name: contract-validate
description: Validate that frontend API calls match the backend API schema specification
tags:
  - api
  - openapi
  - validation
  - contract-testing
trigger: When the user asks to validate API calls, check contract compliance, or find frontend/backend mismatches
---

# Contract Validate

Validate frontend API calls against the backend OpenAPI/Swagger schema. Detects endpoints called by the frontend that don't exist in the spec, checks request/response shapes, and reports mismatches.

## Steps

### 1. Find API Schema

Search the project for the API specification file in this priority order:

1. `openapi.yaml` or `openapi.yml` in the project root
2. `swagger.json` in the project root
3. `openapi.json` in the project root
4. `api-spec.yaml` or `api-spec.yml` in the project root
5. `docs/openapi.yaml`, `docs/openapi.yml`, `docs/swagger.json`, `docs/openapi.json`
6. Any file matching `**/openapi.{yaml,yml,json}` or `**/swagger.{yaml,yml,json}`

If no schema is found, report:
```
No API schema found.
Expected one of: openapi.yaml, swagger.json, openapi.json, api-spec.yaml
Searched in: project root and docs/
```

Parse the schema and extract:
- All defined paths and their HTTP methods
- Request body schemas (`requestBody.content['application/json'].schema`)
- Response schemas per status code
- Required fields on request bodies and response objects
- `$ref` definitions — resolve them before comparing

### 2. Find Frontend API Calls

Scan `src/` (fall back to project root if no `src/`) for all `.ts`, `.tsx`, `.js`, `.jsx` files. Search each for:

- `fetch(` — native fetch. Extract URL string and method from options object (default: GET).
- `axios.get/post/put/delete/patch(` or `axios({ method, url })` — extract method and URL.
- `useSWR(` — extract the key/URL argument.
- `useQuery(` / `useMutation(` — extract query key or URL string.
- `trpc.<router>.<procedure>.useQuery/useMutation(` — map tRPC path to REST equivalent if schema covers it.
- HTTP client instance calls: `api.get(`, `client.post(`, `http.delete(` — extract method and URL.
- Template literals containing `/api/` — extract the path pattern.

For each call record:
- File path and line number
- HTTP method
- URL path or template pattern
- Request body fields (if an object literal is passed inline)
- Response fields read by the caller (destructuring or property access)

### 3. Normalize Endpoints

Normalize discovered calls to `{ method, path }` pairs:

- Strip base URL prefixes (`https://api.example.com`, `process.env.API_URL`, `/api`)
- Convert dynamic segments to `{param}` notation:
  - `` `/users/${id}` `` → `/users/{id}`
  - `'/users/' + userId` → `/users/{id}`
  - `/users/42` (literal number) → `/users/{id}`
- Match normalized paths against spec paths using longest-prefix matching.

### 4. Compare Against Schema

For each frontend endpoint:

| Status | Condition |
|--------|-----------|
| **MATCHED** | Path and method both exist in spec |
| **METHOD_MISMATCH** | Path exists in spec but not with this HTTP method |
| **UNMATCHED** | Path does not exist in spec at all |

Separately collect: spec endpoints never called by the frontend → **UNUSED_SCHEMA_ENDPOINT**.

### 5. Check Request Body Shape

For matched POST/PUT/PATCH endpoints:

1. Resolve the request body schema (follow `$ref` chains).
2. Extract `required[]` and `properties` from the JSON schema.
3. Compare against fields the frontend sends:
   - **MISSING_REQUIRED_FIELD**: frontend omits a field the schema marks required.
   - **EXTRA_FIELD**: frontend sends a field not defined in schema properties (warn only).
   - **TYPE_MISMATCH**: frontend sends a value whose inferred type differs from the schema type.

### 6. Check Response Usage

For matched endpoints, look at what response fields the frontend reads:

1. Identify property accesses like `response.data.items`, `result.user.email`.
2. Resolve the 200 response schema from the spec (follow `$ref`).
3. For each accessed path, verify the field exists in the schema:
   - **MISSING_RESPONSE_FIELD**: frontend reads `response.data.items` but schema has no `items` field.

### 7. Report

Output a structured validation report:

```
API Contract Validation Report
================================

Schema: openapi.yaml (OpenAPI 3.0.3)
Frontend files scanned: 24
Endpoints found in frontend: 11
Endpoints defined in schema: 15

MATCHED (8)
  GET  /users             src/api/users.ts:12
  GET  /users/{id}        src/components/UserProfile.tsx:34
  POST /users             src/pages/CreateUser.tsx:67
  ...

UNMATCHED — frontend calls endpoint not in schema (2)
  POST /users/{id}/avatar  src/components/AvatarUpload.tsx:19
    → No matching path in schema. Did you mean POST /users/{id}/photo?
  DELETE /sessions         src/auth/logout.ts:8
    → Not defined in schema.

METHOD_MISMATCH (1)
  PUT /users/{id}          src/components/EditUser.tsx:45
    → Schema defines PATCH /users/{id}, not PUT.

FIELD MISMATCHES (1)
  POST /users (src/pages/CreateUser.tsx:67)
    → Missing required field: "role" (schema requires: name, email, role)
    → Extra field sent: "displayName" (not in schema properties)

UNUSED SCHEMA ENDPOINTS (4)
  GET  /users/{id}/permissions
  POST /auth/refresh
  GET  /audit-logs
  DELETE /users/{id}

Summary: 3 issues require attention (2 unmatched, 1 field mismatch)
```

**Guardrail**: This skill is read-only. It never modifies source files or the schema.
