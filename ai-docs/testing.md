# Testing

## Two test layers

| Layer | Tool | Location | Runs |
|---|---|---|---|
| Unit / integration | Jest + ts-jest | `src/**/__tests__/*.test.ts` | `npm test` |
| End-to-end | Playwright | `e2e/*.spec.ts` | `npm run test:e2e` |

---

## Unit tests (Jest)

### Config

- `jest.config.js` — ts-jest preset, `node` environment, `@/` alias mapped
- `jest.integration.config.js` — for tests that need a real DB
- `jest.setup.js` — global mocks (Redis, DB, etc.)

### File location convention

```
src/app/api/chores/__tests__/chores.test.ts
src/lib/auth/__tests__/session.test.ts
src/lib/integrations/__tests__/calendar.test.ts
```

Place `__tests__/` directly beside the code being tested.

### Anatomy of a route unit test

```ts
import { POST } from '../route';
import { db } from '@/lib/db/client';
import { requireAuth } from '@/lib/auth';

jest.mock('@/lib/db/client');
jest.mock('@/lib/auth');

const mockDb = db as jest.Mocked<typeof db>;
const mockRequireAuth = requireAuth as jest.MockedFunction<typeof requireAuth>;

describe('POST /api/chores', () => {
  beforeEach(() => {
    mockRequireAuth.mockResolvedValue({ userId: 'user-1', role: 'parent' });
  });

  it('creates a chore', async () => {
    const newChore = { id: 'chore-1', title: 'Dishes', category: 'dishes' };
    mockDb.insert.mockReturnValue({
      values: jest.fn().mockReturnValue({ returning: jest.fn().mockResolvedValue([newChore]) }),
    } as never);

    const request = new Request('http://localhost/api/chores', {
      method: 'POST',
      body: JSON.stringify({ title: 'Dishes', category: 'dishes', frequency: 'daily' }),
      headers: { 'Content-Type': 'application/json' },
    });

    const response = await POST(request as never);
    expect(response.status).toBe(201);
    const body = await response.json();
    expect(body.title).toBe('Dishes');
  });

  it('returns 401 when not authenticated', async () => {
    const { NextResponse } = await import('next/server');
    mockRequireAuth.mockResolvedValue(NextResponse.json({ error: 'Unauthorized' }, { status: 401 }));

    const request = new Request('http://localhost/api/chores', {
      method: 'POST',
      body: JSON.stringify({ title: 'Dishes', category: 'dishes', frequency: 'daily' }),
      headers: { 'Content-Type': 'application/json' },
    });

    const response = await POST(request as never);
    expect(response.status).toBe(401);
  });
});
```

### What to test

- Happy path: correct input → correct DB call → correct response shape
- Auth: unauthenticated → 401; wrong role → 403
- Validation: invalid body → 400 with error details
- Edge cases: entity not found → 404; duplicate → 409

### Mocking Redis / DB

`jest.setup.js` provides global mocks for Redis and the DB client.
Override them per-test with `mockResolvedValue` or `mockReturnValue`.

---

## E2E tests (Playwright)

### Config

`playwright.config.ts` — single Chromium project, base URL `http://localhost:3000`, starts `npm run dev` if not already running.

### File location

```
e2e/
  chores.spec.ts
  chore-crud.spec.ts
  auth.spec.ts
  helpers/
    auth.ts        # login helpers
    ...
```

### Anatomy

```ts
import { test, expect } from '@playwright/test';
import { loginAsParent } from './helpers/auth';

test.describe('Chores', () => {
  test.beforeEach(async ({ page }) => {
    await loginAsParent(page);
    await page.goto('/chores');
  });

  test('can add a chore', async ({ page }) => {
    await page.getByRole('button', { name: 'Add chore' }).click();
    await page.getByLabel('Title').fill('Vacuum living room');
    await page.getByRole('button', { name: 'Save' }).click();
    await expect(page.getByText('Vacuum living room')).toBeVisible();
  });
});
```

### Login helpers

See `e2e/helpers/` for reusable login flows. Parent PIN is `1234`, child PIN is `0000` in the seeded dev database.

### Running

```bash
npm run test:e2e              # headless
npm run test:e2e:ui           # interactive UI mode
npm run test:visual           # visual regression
npm run test:visual:update    # update snapshots
```

---

## Running the full test suite

```bash
# Unit tests
npm test

# Unit tests with coverage
npm run test:coverage

# E2E (starts dev server automatically)
npm run test:e2e
```

---

## CI

GitHub Actions runs both suites on every push. See `.github/workflows/`.
The `forbidOnly` flag in `playwright.config.ts` ensures `test.only` is never committed.
