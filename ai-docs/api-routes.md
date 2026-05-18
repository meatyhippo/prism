# API Route Conventions

## File locations

All API routes live under `src/app/api/` following Next.js App Router conventions:

```
src/app/api/
  chores/
    route.ts              # GET /api/chores, POST /api/chores
    [id]/
      route.ts            # GET /api/chores/:id, PUT /api/chores/:id, DELETE /api/chores/:id
      complete/
        route.ts          # POST /api/chores/:id/complete
```

## Anatomy of a route file

```ts
import { NextRequest, NextResponse } from 'next/server';
import { withAuth } from '@/lib/api/withAuth';
import { db } from '@/lib/db/client';
import { chores } from '@/lib/db/schema';
import { createChoreSchema } from '@/lib/validations';
import { logActivity } from '@/lib/services/auditLog';
import { logError } from '@/lib/utils/logError';

// GET /api/chores — list (any logged-in user)
export async function GET() {
  const auth = await requireAuth();
  if (auth instanceof NextResponse) return auth;

  const rows = await db.select().from(chores);
  return NextResponse.json(rows);
}

// POST /api/chores — create (parent only)
export async function POST(request: NextRequest) {
  return withAuth(async (auth) => {
    const body = await request.json();
    const parsed = createChoreSchema.safeParse(body);
    if (!parsed.success) {
      return NextResponse.json({ error: parsed.error.flatten() }, { status: 400 });
    }

    const [chore] = await db.insert(chores).values(parsed.data).returning();

    logActivity({
      userId: auth.userId,
      action: 'chore.create',
      entityType: 'chore',
      entityId: chore.id,
      summary: `Created chore "${chore.title}"`,
    });

    return NextResponse.json(chore, { status: 201 });
  }, { permission: 'canManageChores' });
}
```

## Auth patterns summary

| Scenario | Pattern |
|---|---|
| Any logged-in user | `requireAuth()` + early return check |
| Role/permission required | `withAuth(handler, { permission: 'canManageChores' })` |
| Optional (works for guests) | `optionalAuth()` — returns `null` if unauthenticated |
| Machine-to-machine only | `withAuth(handler, { tokenScope: 'voice' })` |

## Input validation

Always use Zod schemas from `src/lib/validations/index.ts` for request bodies.
Never access `request.json()` without validation.

```ts
const parsed = createChoreSchema.safeParse(body);
if (!parsed.success) {
  return NextResponse.json({ error: parsed.error.flatten() }, { status: 400 });
}
```

Use `parsed.data` (not `body`) for the rest of the handler.

## Error handling

- Use `logError` from `@/lib/utils/logError` for unexpected server errors.
- Return `{ error: string }` JSON with an appropriate HTTP status.
- 400 — invalid input
- 401 — not authenticated
- 403 — forbidden (role / CSRF)
- 404 — entity not found
- 409 — conflict (e.g. duplicate external ID)
- 500 — unexpected error

```ts
try {
  // ...
} catch (error) {
  logError('Chore create failed:', error);
  return NextResponse.json({ error: 'Failed to create chore' }, { status: 500 });
}
```

## Dynamic route parameters

```ts
export async function GET(
  _request: NextRequest,
  { params }: { params: Promise<{ id: string }> }
) {
  const { id } = await params;
  // validate id is a UUID
  if (!z.string().uuid().safeParse(id).success) {
    return NextResponse.json({ error: 'Invalid id' }, { status: 400 });
  }
  // ...
}
```

## Rate limiting

Available through `withAuth` options. Backed by Redis.

```ts
withAuth(handler, {
  rateLimit: { feature: 'shopping-scan', limit: 30, windowSeconds: 60 },
});
```

Use conservative limits on expensive endpoints (barcode scans, calendar syncs, weather calls).

## Request correlation

`src/middleware.ts` injects a `x-request-id` header on every API request.
Access it for logging when needed:

```ts
const requestId = request.headers.get('x-request-id');
```

## Response conventions

- Lists return a plain array: `NextResponse.json(rows)`
- Single objects return the object directly: `NextResponse.json(chore)`
- Created resources return 201: `NextResponse.json(chore, { status: 201 })`
- Successful mutation with no body: `NextResponse.json({ success: true })`
- Deletion: `NextResponse.json({ success: true })` (200, not 204, for consistency with client fetch)

## Credential access pattern

OAuth credentials are stored either in the `settings` DB table or in env vars. Always use `credentialStore.ts`, never `process.env` directly:

```ts
import { getGoogleCredentials } from '@/lib/integrations/credentialStore';

const creds = await getGoogleCredentials();
if (!creds) {
  return NextResponse.json({ error: 'Google not configured' }, { status: 503 });
}
```
