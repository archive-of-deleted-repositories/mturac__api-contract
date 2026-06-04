# api-contract

Claude Code plugin that validates frontend API calls against backend schemas, detects breaking changes before deploy, and generates typed TypeScript clients.

## Install

```bash
claude plugin install mturac/api-contract
```

Or from local path:

```bash
claude plugin install /path/to/api-contract
```

## Commands

| Command | What it does |
|---|---|
| `/api-contract` | Run full validation (schema + frontend calls + field checks) |
| `/api-contract-validate` | Validate frontend calls against schema |
| `/api-contract-diff [ref1] [ref2]` | Find breaking changes between two spec versions |
| `/api-contract-generate` | Generate TypeScript types and fetch client from spec |

## Example: Validation Output

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
  PUT  /users/{id}        src/components/EditUser.tsx:45
  GET  /products          src/pages/Products.tsx:18
  GET  /products/{id}     src/pages/ProductDetail.tsx:22
  POST /orders            src/pages/Checkout.tsx:91
  GET  /orders/{id}       src/pages/OrderDetail.tsx:14

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

UNUSED SCHEMA ENDPOINTS (4)
  GET  /users/{id}/permissions
  POST /auth/refresh
  GET  /audit-logs
  DELETE /users/{id}

Summary: 3 issues require attention (2 unmatched, 1 field mismatch)
```

## Example: Breaking Change Detection

```
API Contract Diff: main → HEAD
Schema file: openapi.yaml
================================

BREAKING CHANGES (2)
  CRITICAL  DELETE /users/{id}        Endpoint removed
  HIGH      POST /users → body.role   Field added to required[]

ADDITIVE CHANGES (1)
  +  GET /users/{id}/permissions      New endpoint

Summary: 2 breaking changes detected. Review before deploying.
```

## Example: Generated TypeScript

```ts
// src/api/types.ts (auto-generated)
export interface CreateUserRequest {
  name: string;
  email: string;
  role: 'admin' | 'user' | 'viewer';
}

export interface User {
  id: string;
  name: string;
  email: string;
  role: 'admin' | 'user' | 'viewer';
  createdAt: string;
}

// src/api/client.ts (auto-generated)
export async function createUser(
  body: CreateUserRequest,
  options?: RequestInit
): Promise<User> { ... }

export async function getUserById(
  params: { id: string },
  options?: RequestInit
): Promise<User> { ... }
```

## Schema discovery

Looks for API specs in this order:
1. `openapi.yaml` / `openapi.yml` (project root)
2. `swagger.json` (project root)
3. `openapi.json` (project root)
4. `api-spec.yaml` (project root)
5. `docs/openapi.yaml`, `docs/swagger.json`

Supports OpenAPI 3.x and Swagger 2.x.
