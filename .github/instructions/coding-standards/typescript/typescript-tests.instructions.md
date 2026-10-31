---
description: 'Required instructions for TypeScript test code research, planning, implementation, editing, or creating'
applyTo: '**/*.test.ts, **/*.test.tsx, **/*.spec.ts, **/*.test.mts, **/*.spec.mts, **/*.test.cts, **/*.spec.cts'
---

# TypeScript Test Instructions

All conventions from [typescript.instructions.md](typescript.instructions.md) apply, including strict typing, no `any`, and `import type`.

> Versions: repo pinning targets Vitest 3.x, `@cloudflare/vitest-pool-workers` 0.23.x, and Playwright 1.x. Keep the major versions aligned with the lockfile before upgrading the toolchain.

## Test Framework

| Framework | Usage |
|-----------|-------|
| Vitest | Unit and integration tests; default for all non-handler tests |
| `@cloudflare/vitest-pool-workers` | Pages Function and Worker handler tests requiring real D1, KV, or other bindings |
| Playwright | End-to-end tests only; never for unit or integration |

Use `restoreMocks: true` and `mockReset: true` in `vitest.config.ts` to reset mock state between tests.

## Test Placement

Place unit and integration tests in `tests/`, mirroring `src/` with a `.test.ts` suffix.

<!-- <example-test-placement> -->
```text
src/functions/api/posts.ts        →  tests/functions/api/posts.test.ts
src/lib/validators.ts             →  tests/lib/validators.test.ts
```
<!-- </example-test-placement> -->

Place Playwright end-to-end tests in `e2e/` with a `.spec.ts` suffix. Never mix unit and E2E tests in the same directory.

## Test Naming

Name each `it` or `test` with a complete sentence describing the observed behavior.

<!-- <example-test-naming> -->
```text
returns 400 when request body is not valid JSON
returns 422 with field errors when a required property is missing
returns 201 with a UUID post id on a valid request
throws when the database connection is unavailable
```
<!-- </example-test-naming> -->

Cover one behavior per `it` or `test`. Do not combine assertions for unrelated behaviors in a single test.

## Test Organization

- Group related tests with `describe` named after the function or module under test
- Name the system under test `sut` when the value is a describe-scoped instance constructed in `beforeEach`; inline per-test values do not need the `sut` name
- Use `beforeEach` for setup that must be fresh per test; use `beforeAll` for one-time setup shared by all tests (for example, database schema creation)
- Keep mock reset in `vitest.config.ts`; do not add a second layer of per-test mock cleanup unless a test needs a custom reset
- Never share mutable state between tests via outer-scope variables without resetting in `beforeEach`

<!-- <example-test-organization> -->
```typescript
describe('UserService', () => {
  let sut: UserService;

  beforeEach(() => {
    sut = new UserService(makeMockDb());
  });

  it('returns null when the user does not exist', async () => {
    const result = await sut.findUser('missing-id');
    expect(result).toBeNull();
  });
});
```
<!-- </example-test-organization> -->

## Vitest Mock Patterns

- Use `vi.fn()` with an explicit function-type generic for standalone mocks: `vi.fn<(arg: string) => Promise<User | null>>()`
- Use `vi.spyOn(object, 'method')` to replace a single method while keeping others intact
- Call `vi.mock('@/lib/module')` at file scope, never inside a test or lifecycle hook
- Pair `vi.useFakeTimers()` with `vi.useRealTimers()` in `afterEach`
- Prefer injected fake objects over `vi.mock` when the dependency is passed as a parameter — injection is faster, scoped to the test, and does not require module reloading
- Keep `restoreMocks: true` and `mockReset: true` in `vitest.config.ts`; do not add `vi.restoreAllMocks()` or `vi.mockReset()` calls in each test unless a custom reset is truly required

<!-- <example-mock-patterns> -->
```typescript
// Injected fake — preferred for passed dependencies
const fetchMock = vi.fn<(url: string) => Promise<Response>>();
fetchMock.mockResolvedValue(new Response(JSON.stringify({ id: '1' }), { status: 200 }));

// Module mock — only when injection is not possible
vi.mock('@/lib/db', () => ({ getDb: vi.fn() }));

// Spy with per-test restore
const spy = vi.spyOn(console, 'error').mockImplementation(() => undefined);
afterEach(() => spy.mockRestore());

// Fake timers
beforeEach(() => vi.useFakeTimers());
afterEach(() => vi.useRealTimers());
```
<!-- </example-mock-patterns> -->

## Test Data

- Use typed factory functions to build test fixtures; do not repeat inline object literals across tests
- Derive fixture types from Zod schemas with `z.infer<typeof Schema>` — never define a parallel interface in the test file
- Use `Partial<T>` spread for factory overrides to keep fixtures minimal

<!-- <example-test-data> -->
```typescript
import { z } from 'zod';
import { CreatePostSchema } from '@/functions/api/posts';

type CreatePost = z.infer<typeof CreatePostSchema>;

function postFixture(overrides: Partial<CreatePost> = {}): CreatePost {
  return { title: 'Test Post', content: 'Test content', ...overrides };
}
```
<!-- </example-test-data> -->

## Handler Tests

- Construct a `Request` with the method, headers, and body matching the scenario under test
- Add `tests/env.d.ts` with `declare module 'cloudflare:test' { interface ProvidedEnv extends Env {} }` so `env` includes your Worker bindings
- Use `env` from `cloudflare:test` when real bindings (D1, KV) are needed; this requires `@cloudflare/vitest-pool-workers`
- Define the required D1 binding in `wrangler.toml` with `[[d1_databases]]` and `binding = "DB"`
- Build an `EventContext` with `satisfies EventContext<typeof env, path, unknown>` to validate the shape without a type assertion
- Assert on both `response.status` and the parsed body — never assert on status alone
- Parse `response.json()` into a variable typed as `unknown`, then assert with `toMatchObject` or Zod

Configure `@cloudflare/vitest-pool-workers` in `vitest.config.ts` and run D1 migrations from a setup file:

<!-- <example-vitest-config> -->
```typescript
// vitest.config.ts
import { defineWorkersConfig } from '@cloudflare/vitest-pool-workers/config';

export default defineWorkersConfig({
  test: {
    restoreMocks: true,
    mockReset: true,
    setupFiles: ['./tests/setup-d1.ts'],
    poolOptions: {
      workers: {
        wrangler: { configPath: './wrangler.toml' },
      },
    },
  },
});
```
<!-- </example-vitest-config> -->

```toml
# wrangler.toml
[[d1_databases]]
binding = "DB"
database_name = "my-app"
database_id = "00000000-0000-0000-0000-000000000000"
```

```ts
// tests/env.d.ts
declare module 'cloudflare:test' {
  interface ProvidedEnv extends Env {}
}
```

```ts
// tests/setup-d1.ts
import { env, applyD1Migrations } from 'cloudflare:test';
import { beforeAll } from 'vitest';
import { readD1Migrations } from '@cloudflare/vitest-pool-workers/config';

beforeAll(async () => {
  await applyD1Migrations(env.DB, await readD1Migrations('./migrations'));
});
```

<!-- <example-handler-tests> -->
```typescript
import { env } from 'cloudflare:test';
import type { EventContext } from '@cloudflare/workers-types';

function postContext(
  body: unknown,
  options: { malformed?: boolean } = {},
): EventContext<typeof env, '/api/posts', unknown> {
  return {
    request: new Request('https://example.com/api/posts', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: options.malformed ? '{bad-json' : JSON.stringify(body),
    }),
    env,
    params: {},
    data: {},
    functionPath: '/api/posts',
    waitUntil: () => undefined,
    passThroughOnException: () => undefined,
    next: () => Promise.resolve(new Response(null, { status: 404 })),
  };
}
```
<!-- </example-handler-tests> -->

## E2E Tests (Playwright)

- Use role- and label-based locators (`getByRole`, `getByLabel`, `getByText`); never use CSS selectors or XPath (`@playwright/no-raw-locators`)
- Never use `page.waitForTimeout` or arbitrary delays; Playwright's `expect` assertions auto-retry (`@playwright/no-wait-for-timeout`)
- Cover one user flow per test; do not chain unrelated actions in a single test (reviewer-checked; no dedicated `eslint-plugin-playwright` rule)

<!-- <example-e2e> -->
```typescript
import { test, expect } from '@playwright/test';

test('user can create a post', async ({ page }) => {
  await page.goto('/posts/new');
  await page.getByLabel('Title').fill('My Post');
  await page.getByLabel('Content').fill('Hello world');
  await page.getByRole('button', { name: 'Publish' }).click();
  await expect(page.getByRole('heading', { name: 'My Post' })).toBeVisible();
});
```
<!-- </example-e2e> -->

## Test Conventions

- Every test must include at least one assertion (`vitest/expect-expect`)
- Never commit focused tests (`it.only`, `describe.only`) to the repository (`vitest/no-focused-tests`)
- Never commit skipped tests (`it.skip`, `describe.skip`) without a reviewer-checked reason in a comment directly above the test; when linting is enabled, the project enforces `vitest/no-disabled-tests`
- Do not use `expect.assertions(n)` as a substitute for explicit assertions in async tests
- Each test must pass in isolation regardless of execution order

## Complete Example

Demonstrates: `@cloudflare/vitest-pool-workers` with real D1 bindings, `tests/env.d.ts` providing `ProvidedEnv`, a D1 migration setup file backed by `readD1Migrations` + `applyD1Migrations`, one behavior per `it`, `body: unknown` from `response.json()`, and `toMatchObject` for partial shape assertion.

<!-- <example-complete> -->
```typescript
// tests/functions/api/posts.test.ts
// Requires @cloudflare/vitest-pool-workers; see vitest.config.ts
import { describe, it, expect, beforeAll } from 'vitest';
import { env, applyD1Migrations } from 'cloudflare:test';
import { readD1Migrations } from '@cloudflare/vitest-pool-workers/config';
import { onRequestPost } from '@/functions/api/posts';
import type { EventContext } from '@cloudflare/workers-types';

// tests/env.d.ts
// declare module 'cloudflare:test' {
//   interface ProvidedEnv extends Env {}
// }

// wrangler.toml
// [[d1_databases]]
// binding = "DB"
// database_name = "my-app"
// database_id = "00000000-0000-0000-0000-000000000000"

// ── Fixture builder ───────────────────────────────────────────────────────────
function postContext(
  body: unknown,
  options: { malformed?: boolean } = {},
): EventContext<typeof env, '/api/posts', unknown> {
  return {
    request: new Request('https://example.com/api/posts', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: options.malformed ? '{bad-json' : JSON.stringify(body),
    }),
    env,
    params: {},
    data: {},
    functionPath: '/api/posts',
    waitUntil: () => undefined,
    passThroughOnException: () => undefined,
    next: () => Promise.resolve(new Response(null, { status: 404 })),
  };
}

// ── One-time schema setup ─────────────────────────────────────────────────────
beforeAll(async () => {
  await applyD1Migrations(env.DB, await readD1Migrations('./migrations'));
});

// ── Tests ─────────────────────────────────────────────────────────────────────
describe('onRequestPost', () => {
  it('returns 400 when request body is not valid JSON', async () => {
    const response = await onRequestPost(postContext({}, { malformed: true }));

    expect(response.status).toBe(400);
    const body: unknown = await response.json();
    expect(body).toMatchObject({ ok: false, errors: [{ field: 'body' }] });
  });

  it('returns 422 with field errors when title is missing', async () => {
    const response = await onRequestPost(postContext({ content: 'Hello' }));

    expect(response.status).toBe(422);
    const body: unknown = await response.json();
    expect(body).toMatchObject({ ok: false, errors: [{ field: 'title' }] });
  });

  it('returns 201 with a UUID post id on a valid request', async () => {
    const response = await onRequestPost(
      postContext({ title: 'My Post', content: 'Hello world' }),
    );

    expect(response.status).toBe(201);
    const body: unknown = await response.json();
    expect(body).toMatchObject({
      ok: true,
      data: { id: expect.stringMatching(/^[0-9a-f]{8}-(?:[0-9a-f]{4}-){3}[0-9a-f]{12}$/) },
    });
  });
});
```
<!-- </example-complete> -->

This example demonstrates:

- `tests/env.d.ts` providing `ProvidedEnv extends Env` so the Worker `DB` binding is available through `env`
- the `wrangler.toml` D1 binding (`binding = "DB"`) required for real integration tests
- `applyD1Migrations(env.DB, await readD1Migrations('./migrations'))` in a test setup file for schema creation
- `EventContext<typeof env, '/api/posts', unknown>` for typed context construction without an unsafe cast
- One `it` block per behavior (400, 422, 201) with independent state
- `body: unknown = await response.json()`; `any` is never used
- `toMatchObject` for partial shape assertion on the discriminated union response
