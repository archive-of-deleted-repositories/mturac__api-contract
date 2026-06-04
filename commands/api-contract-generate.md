# /api-contract-generate

Generate TypeScript interfaces and typed fetch wrapper functions from an OpenAPI/Swagger spec. Outputs `src/api/types.ts` and `src/api/client.ts`.

## Usage

```
/api-contract-generate
/api-contract-generate --schema path/to/openapi.yaml
/api-contract-generate --out src/lib/api
```

## Options

- `--schema <path>` — override schema file path (default: auto-discover openapi.yaml / swagger.json)
- `--out <dir>` — output directory for generated files (default: `src/api/`)
- `--no-client` — only generate `types.ts`, skip `client.ts`
- `--force` — overwrite manually-edited files without creating `.bak` backup

## What gets generated

### `src/api/types.ts`
TypeScript interfaces for every schema in `components/schemas`, plus request/response types per endpoint. Enums become string literal unions. Nullable fields get `| null`. Optional fields use `?`.

### `src/api/client.ts`
One async function per endpoint using native `fetch`. Each function is fully typed: path params, query params, and request body are all type-checked. Non-2xx responses throw an error with status code.

```ts
// Example generated function
export async function createUser(
  body: CreateUserRequest,
  options?: RequestInit
): Promise<CreateUserResponse> { ... }

export async function getUserById(
  params: { id: string },
  options?: RequestInit
): Promise<GetUserByIdResponse> { ... }
```

## Notes

- Generated files include a "DO NOT EDIT" header and generation timestamp.
- If an existing file lacks this header, a `.bak` copy is created before overwriting.
- The client uses `NEXT_PUBLIC_API_URL` or `VITE_API_URL` env vars for the base URL.

## Skill

Executes: `skills/contract-generate/SKILL.md`
