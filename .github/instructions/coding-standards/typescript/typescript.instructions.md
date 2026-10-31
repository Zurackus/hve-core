---
description: 'Required instructions for TypeScript research, planning, implementation, editing, or creating'
applyTo: '**/*.ts, **/*.tsx, **/*.mts, **/*.cts'
---

# TypeScript Instructions

Conventions for TypeScript development targeting TypeScript 5.x with strict type-safety on Node.js 20+ and Cloudflare edge runtimes.

## Project Structure

<!-- <example-project-structure> -->
```text
pnpm-workspace.yaml
package.json
tsconfig.json
eslint.config.mjs
src/
  index.ts
  functions/
    api/
  lib/
  types/
tests/
```
<!-- </example-project-structure> -->

- `src/` contains all production TypeScript source
- `tests/` mirrors the `src/` directory structure
- Path aliases configured in `tsconfig.json` `paths`; `@/*` maps to `src/*`

## Configuration

| Option | Value | Effect |
|--------|-------|--------|
| `strict` | `true` | Enables all strict checks |
| `noUncheckedIndexedAccess` | `true` | Array/record indexing returns `T \| undefined` |
| `exactOptionalPropertyTypes` | `true` | Distinguishes `{ x?: T }` from `{ x: T \| undefined }` |
| `verbatimModuleSyntax` | `true` | Enforces `import type` for type-only imports |
| `moduleResolution` | `"bundler"` | Matches Vite/esbuild; allows `.ts` extensions |

<!-- <example-tsconfig> -->
```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "verbatimModuleSyntax": true,
    "moduleResolution": "bundler",
    "module": "ESNext",
    "target": "ESNext",
    "paths": { "@/*": ["./src/*"] }
  }
}
```
<!-- </example-tsconfig> -->

<!-- <example-eslint-config> -->
```js
// eslint.config.mjs (typescript-eslint strictTypeChecked + stylisticTypeChecked + import rules)
import tseslint from 'typescript-eslint';
import importPlugin from 'eslint-plugin-import';

export default tseslint.config(
  tseslint.configs.strictTypeChecked,
  tseslint.configs.stylisticTypeChecked,
  importPlugin.flatConfigs.recommended,
  {
    languageOptions: {
      parserOptions: {
        projectService: true,
      },
    },
    rules: {
      '@typescript-eslint/consistent-type-imports': ['error', { prefer: 'type-imports' }],
      'import/no-default-export': 'error',
      'import/order': [
        'error',
        {
          groups: ['builtin', 'external', 'internal', 'parent', 'sibling', 'index'],
          pathGroups: [
            { pattern: '@/*', group: 'internal' },
          ],
          alphabeticalOrder: true,
          caseInsensitive: true,
        },
      ],
      'no-restricted-syntax': [
        'error',
        {
          selector: 'TSEnumDeclaration',
          message: 'Use `as const` with union types instead of enums',
        },
      ],
    },
  },
);
```
<!-- </example-eslint-config> -->

<!-- <example-prettier-config> -->
.prettierrc:
```json
{ "semi": false, "singleQuote": true, "trailingComma": "all" }
```
<!-- </example-prettier-config> -->

## Managing Projects

<!-- <example-cli-commands> -->
```bash
pnpm install                    # install dependencies
pnpm add zod                    # add production dependency
pnpm add -D typescript vitest   # add dev dependencies
pnpm tsc --noEmit               # type-check without emitting
pnpm eslint src                 # lint TypeScript source
pnpm vitest run                 # run tests once (CI)
wrangler types --env-interface CloudflareBindings  # generate typed bindings
```
<!-- </example-cli-commands> -->

## Coding Conventions

| Element | Convention | Example |
|---------|------------|---------|
| Files | `kebab-case` | `user-service.ts` |
| Classes / Interfaces / Types | `PascalCase` | `UserService`, `ApiResponse` |
| Functions / Variables | `camelCase` | `fetchUser`, `isActive` |
| Module-level constants | `SCREAMING_SNAKE_CASE` | `MAX_RETRIES` |
| React / Astro components | `PascalCase` file | `UserCard.astro`, `PostList.tsx` |
| Environment variables | `SCREAMING_SNAKE_CASE` | `DATABASE_URL` |
| Type parameters | `T` prefix | `TData`, `TError` |

### Import Ordering

Enforced by `import/order`. Order:

1. Node built-ins (`node:fs`, `node:path`) — avoid in edge runtime code
2. External packages
3. Internal path aliases (`@/`)
4. Relative imports (`./`, `../`)

Use `import type` for all type-only imports. `verbatimModuleSyntax` and `@typescript-eslint/consistent-type-imports` enforce this.

### No Enums

Never use `enum` (`no-restricted-syntax` with selector `TSEnumDeclaration`). Use `as const` with union types instead.

<!-- <example-no-enums> -->
```typescript
// ❌ enum erases to IIFE — causes bundle bloat and barrel-file issues
enum Status { Active = 'active', Inactive = 'inactive' }

// ✅
const Status = { Active: 'active', Inactive: 'inactive' } as const;
type Status = (typeof Status)[keyof typeof Status];
```
<!-- </example-no-enums> -->

### Avoid `any`

Never use `any` (`@typescript-eslint/no-explicit-any`). Use `unknown` and narrow at usage sites.

<!-- <example-avoid-any> -->
```typescript
// ❌
function process(data: any): void {}

// ✅
function process(data: unknown): void {
  if (typeof data !== 'string') throw new TypeError('Expected string');
  // data is now string
}
```
<!-- </example-avoid-any> -->

### Discriminated Unions

Use discriminated unions instead of nullable return types for domain operations. Enable exhaustive switch checking with `@typescript-eslint/switch-exhaustiveness-check`.

<!-- <example-discriminated-union> -->
```typescript
type Result<T, E = string> =
  | { readonly ok: true; readonly value: T }
  | { readonly ok: false; readonly error: E };
```
<!-- </example-discriminated-union> -->

### Readonly by Default

- Apply `readonly` to all interface and type properties unless mutation is intentional
- Use `readonly T[]` for function parameters accepting arrays (matches stylistic array-type)
- `@typescript-eslint/prefer-readonly` enforces readonly on class properties

### satisfies for Shape Assertion

Use `satisfies` to assert a value matches a type without widening the inferred type. Prefer `satisfies` over type annotation when the narrowest inferred type is needed downstream, and over `as` when you want compile-time safety without a cast.

<!-- <example-satisfies> -->
```typescript
// ❌ type annotation widens — downstream loses the literal inferred type
const body: ApiResponse<never> = { ok: false, errors: [] };

// ❌ as cast bypasses type checking entirely — unsafe
const body = { ok: false, errors: [] } as ApiResponse<never>;

// ✅ satisfies checks shape at compile time; inferred type stays narrow
const body = { ok: false, errors: [] } satisfies ApiResponse<never>;
```
<!-- </example-satisfies> -->

### Exports

Prefer named exports over default exports (`import/no-default-export`). Default exports break auto-import renaming and make grep-based discovery unreliable.

<!-- <example-exports> -->
```typescript
// ❌ default export — import name is arbitrary
export default function createUser() {}

// ✅ named export — import name matches the definition
export function createUser() {}
```
<!-- </example-exports> -->

## Boundary Validation

- Validate with Zod at every trust boundary: HTTP request bodies, environment variables, CMS payloads, third-party API responses
- Derive TypeScript types from schemas with `z.infer<typeof Schema>`; never define a parallel interface
- Use `safeParse` in API handlers to return structured error responses; use `parse` only when an invalid value must throw (for example, startup validation)
- Never use `as` to bypass Zod validation (`@typescript-eslint/no-unsafe-type-assertion`, typescript-eslint v8+)
- Validate environment variables lazily on first request using memoized `parse` of `context.env` (not at module scope), so errors surface at runtime when invoked. For Astro, use `astro:env` for compile-time safety

<!-- <example-boundary-validation> -->
```typescript
import { z } from 'zod';

const CreatePostSchema = z.object({
  title: z.string().min(1).max(100),
  content: z.string().min(1),
});
type CreatePost = z.infer<typeof CreatePostSchema>;

const result = CreatePostSchema.safeParse(rawBody);
if (!result.success) {
  return Response.json({ errors: result.error.issues }, { status: 422 });
}
const post: CreatePost = result.data;
```
<!-- </example-boundary-validation> -->

## Error Handling

- Throw for unrecoverable failures: programming errors, unexpected external failures, broken invariants
- Return a Result union for expected domain failures where the caller should decide how to proceed
- Never swallow errors with an empty `catch` block
- Always wrap re-thrown errors with context: `throw new Error('Context', { cause: err })`
- Log at the outermost handler only (Pages Function handler, top-level route handler, or CLI entry point); do not log inside every catch block in the call stack

<!-- <example-error-handling> -->
```typescript
// ❌
try { await doThing(); } catch (_) {}

// ✅ domain failure — return Result
async function findUser(id: string): Promise<Result<User>> {
  const row = await db.prepare('SELECT * FROM users WHERE id = ?').bind(id).first<User>();
  if (!row) return { ok: false, error: `User ${id} not found` };
  return { ok: true, value: row };
}

// ✅ unexpected failure — throw with cause
try {
  await externalService.call();
} catch (err) {
  throw new Error('External service call failed', { cause: err });
}
```
<!-- </example-error-handling> -->

## Async

- Await all Promise-returning calls; never fire-and-forget (`@typescript-eslint/no-floating-promises`)
- Use `AbortController` and `AbortSignal` for fetch timeouts; do not rely on platform defaults
- Use `Promise.allSettled` when multiple independent operations must all complete regardless of individual failures
- Use `Promise.all` only when all results are required and any failure should abort the batch
- Avoid top-level `await` in shared library modules; use it only in entry points and handlers

<!-- <example-async> -->
```typescript
// AbortSignal timeout
const res = await fetch(url, { signal: AbortSignal.timeout(5_000) });

// Promise.allSettled for independent work
const [postsResult, usersResult] = await Promise.allSettled([
  fetchPosts(),
  fetchUsers(),
]);
for (const result of [postsResult, usersResult]) {
  if (result.status === 'rejected') console.error(result.reason);
}
```
<!-- </example-async> -->

## Edge Runtime Constraints

- Cloudflare Workers and Pages Functions run on V8 isolates; Node.js built-in modules (`fs`, `path`, `crypto`, `os`, `process`) are not available unless `nodejs_compat` is enabled
- Prefer Web APIs regardless: `fetch`, `crypto.randomUUID()`, `URL`, `Request`, `Response`, `Headers`, `TextEncoder`
- Generate typed bindings with `wrangler types` (without `--env-interface`) to obtain a global `Env` type; the output file is `worker-configuration.d.ts`
- Never access secrets via `process.env`; secrets are available only through the `env` parameter in the handler context
- Never import Node.js polyfill packages as a workaround; use the native Web API equivalent

<!-- <example-edge-runtime> -->
```typescript
// ❌ Node crypto unavailable on edge (unless nodejs_compat enabled)
import { randomUUID } from 'crypto';

// ✅ Web Crypto API — available everywhere
const id = crypto.randomUUID();

// ✅ Use global Env type generated by `wrangler types`
export async function onRequestGet(
  context: EventContext<Env, string, unknown>,
): Promise<Response> {
  const cached = await context.env.CACHE.get('data');
  return Response.json({ data: cached });
}
```
<!-- </example-edge-runtime> -->

## Code Documentation

- All exported functions, classes, and types require a TSDoc block
- Use `@param` for every parameter, `@returns` for non-void, `@throws` for thrown errors
- Add `@example` when the function has constraints not captured by the signature (side effects, required call order, non-obvious return shape)
- Use `{@link Symbol}` for cross-references to other types or functions
- Interface implementations use `/** {@inheritDoc Interface.method} */` on overrides
- Never restate the function signature in prose

<!-- <example-tsdoc> -->
```typescript
// ❌ restates the signature — adds no information
/** Gets a user by userId. */
async function getUser(userId: string): Promise<User | null> {}

// ✅ explains what the signature cannot
/**
 * Fetches a user record from the database.
 *
 * Returns `null` when the user does not exist rather than throwing,
 * so callers can branch without a try/catch.
 *
 * @param userId - The UUID assigned at registration.
 * @returns The user record, or `null` if not found.
 * @throws {DatabaseError} When the database connection fails.
 *
 * @example
 * const user = await getUser('user_abc123');
 * if (!user) return notFound();
 */
async function getUser(userId: string): Promise<User | null> {}
```
<!-- </example-tsdoc> -->

## Complete Example

Demonstrates: global typed Cloudflare bindings via `wrangler types`, Zod schema + `z.infer<>`, discriminated union response type with `readonly` properties, `safeParse` at the trust boundary, `unknown` for raw JSON input, `satisfies` for response shape assertion, `import type`, TSDoc on the exported handler, and `crypto.randomUUID()` via the Web Crypto API.

<!-- <example-complete> -->
```typescript
// src/functions/api/posts.ts
import type { EventContext } from '@cloudflare/workers-types';
import { z } from 'zod';

// ── Schema + inferred type ────────────────────────────────────────────────────
const CreatePostSchema = z.object({
  title: z.string().min(1).max(100),
  content: z.string().min(1),
});

type CreatePost = z.infer<typeof CreatePostSchema>;

// ── Discriminated union response type ─────────────────────────────────────────
type ApiError = { readonly field: string; readonly message: string };

type ApiResponse<T> =
  | { readonly ok: true; readonly data: T }
  | { readonly ok: false; readonly errors: readonly ApiError[] };

// ── Domain logic ──────────────────────────────────────────────────────────────
async function insertPost(
  db: D1Database,
  body: CreatePost,
): Promise<{ readonly id: string }> {
  const id = crypto.randomUUID();
  await db
    .prepare('INSERT INTO posts (id, title, content) VALUES (?, ?, ?)')
    .bind(id, body.title, body.content)
    .run();
  return { id };
}

// ── Handler ───────────────────────────────────────────────────────────────────
/**
 * Creates a new post from a validated JSON request body.
 *
 * @param context - The Pages Function execution context with typed bindings from global `Env`.
 * @returns 201 with the new post ID, 400 on malformed JSON, or 422 on schema violations.
 * @throws {Error} When the database insert fails.
 */
export async function onRequestPost(
  context: EventContext<Env, '/api/posts', unknown>,
): Promise<Response> {
  const { request, env } = context;

  let rawBody: unknown;
  try {
    rawBody = await request.json();
  } catch {
    return Response.json(
      { ok: false, errors: [{ field: 'body', message: 'Invalid JSON' }] } satisfies ApiResponse<never>,
      { status: 400 },
    );
  }

  const parsed = CreatePostSchema.safeParse(rawBody);
  if (!parsed.success) {
    const errors = parsed.error.issues.map(
      (issue): ApiError => ({
        field: issue.path.join('.'),
        message: issue.message,
      }),
    );
    return Response.json(
      { ok: false, errors } satisfies ApiResponse<never>,
      { status: 422 },
    );
  }

  const post = await insertPost(env.DB, parsed.data);
  return Response.json(
    { ok: true, data: post } satisfies ApiResponse<{ id: string }>,
    { status: 201 },
  );
}
```
<!-- </example-complete> -->

This example demonstrates:

- Global `Env` bindings type from `wrangler types`, with no secrets accessed via `process.env`
- Zod schema with `z.infer<typeof CreatePostSchema>` for type derivation, replacing any parallel interface
- Discriminated union `ApiResponse<T>` with `readonly` on all properties
- `safeParse` at the HTTP trust boundary, returning structured 422 responses on schema violations
- `unknown` typed `rawBody`; `any` is never used
- `satisfies` for inline response shape assertion without widening the inferred type
- `import type` for the type-only `EventContext` import, enforced by `verbatimModuleSyntax` and `@typescript-eslint/consistent-type-imports`
- TSDoc on the exported handler with `@param` and `@returns` — no restatement of the signature
- `crypto.randomUUID()` via the Web Crypto API, not the Node.js `crypto` module
- Named export only; no default exports
