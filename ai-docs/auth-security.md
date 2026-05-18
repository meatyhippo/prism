# Auth and Security Patterns

## Authentication overview

Prism uses PIN-based family authentication — no passwords, no OAuth accounts on the user record itself.

There are two auth paths:

| Path | Used by |
|---|---|
| Session cookie (`prism_session`) | Browser UI (dashboard, all pages) |
| Bearer token (`Authorization: Bearer …`) | Machine-to-machine (Alexa Voice API, external scripts) |

Both paths converge on the same `requireAuth()` helper, which returns `AuthResult | NextResponse`.

## Roles

Three roles exist:

| Role | Can do |
|---|---|
| `parent` | Everything — manage members, settings, approve chores |
| `child` | Read own data, complete chores, view calendar |
| `guest` | Read-only; no PIN required |

Role permissions are defined centrally in `src/types/user.ts` as `RolePermissions`.

## Using auth in an API route

### Basic (any logged-in user)

```ts
import { requireAuth } from '@/lib/auth';

export async function GET() {
  const auth = await requireAuth();
  if (auth instanceof NextResponse) return auth;   // 401 / 503 returned automatically

  // auth.userId, auth.role available here
}
```

### With a permission check

```ts
import { withAuth } from '@/lib/api/withAuth';

export async function POST(request: NextRequest) {
  return withAuth(async (auth) => {
    // Only parent can reach here
    return NextResponse.json({ ok: true });
  }, { permission: 'canManageChores' });
}
```

`withAuth` handles:
- calling `requireAuth()`
- checking role permissions / token scopes
- applying rate limits (Redis-backed)
- returning well-formed error responses

### Optional auth (public reads)

```ts
import { optionalAuth } from '@/lib/auth';

const auth = await optionalAuth();
// auth is null if not logged in
```

### Token-only endpoint (Voice API)

```ts
return withAuth(async (auth) => { ... }, {
  tokenScope: 'voice',   // rejects session cookies, requires API token with scope
  rateLimit: { feature: 'voice-api', limit: 60, windowSeconds: 60 },
});
```

## Session lifecycle

Sessions are stored in Redis (`session:{token}` key). On every valid request the TTL is refreshed (sliding window).

Key functions in `src/lib/auth/session.ts`:

| Function | Purpose |
|---|---|
| `createSession(userId, role, metadata)` | Create a new session token; returns `null` if Redis is down |
| `validateSession(token)` | Validate + slide window; returns `{ ok, session }` or `{ ok: false, reason }` |
| `invalidateSession(token, userId?)` | Log out one device |
| `invalidateAllUserSessions(userId)` | Log out all devices |
| `recordFailedLogin(userId)` | Increment attempt counter with escalating lockout tiers |
| `clearLoginAttempts(userId)` | Reset after successful login |

Session durations differ by role; defined in `src/lib/constants.ts`.

## CSRF protection

Handled in `src/middleware.ts` — applied to all `POST / PUT / PATCH / DELETE /api/*` requests:

- If `Origin` header present → must match `Host`. Returns 403 otherwise.
- If no `Origin` header → non-browser client, allowed through (relies on bearer token auth layer).
- Paths in `CSRF_EXEMPT_PREFIXES` skip this check.
- `DEMO_MODE=true` blocks all mutations except login/logout.

## Encryption

All OAuth tokens stored in the database are encrypted at rest with AES-256-GCM.

`src/lib/utils/crypto.ts` exports:
- `encrypt(plaintext: string): string` — returns base64 `iv + authTag + ciphertext`
- `decrypt(encoded: string): string`
- `isEncrypted(value: string): boolean` — avoids double-encrypting during migrations

Key source: `ENCRYPTION_KEY` env var (64 hex chars = 32 bytes). Falls back to `PIN_ENCRYPTION_KEY` for legacy installs.

**Always encrypt before writing access/refresh tokens to the DB. Always decrypt before using them.**

```ts
import { encrypt, decrypt } from '@/lib/utils/crypto';

// Writing
await db.update(calendarSources).set({ accessToken: encrypt(rawToken) });

// Reading
const raw = decrypt(source.accessToken!);
```

## Settings PIN gate

Some settings sections require re-entering the parent PIN even when already logged in as a parent. Use `src/lib/auth/settingsAuth.ts` to enforce this.

## API tokens (machine-to-machine)

Generated via `Settings → Security → API Tokens`. Each token has an array of scopes (e.g. `['voice']` or `['*']`).

Tokens are stored as bcrypt hashes. The raw token is only shown once at creation.

`src/lib/auth/apiTokens.ts` exports:
- `generateApiToken()` — creates a raw token string
- `hashToken(raw)` — bcrypt hash for storage
- `validateApiToken(raw)` — returns `ApiTokenAuthResult | null`
- `createApiToken / revokeApiToken / listApiTokens` — CRUD

## Security headers

Applied globally via `next.config.js` → `buildSecurityHeaders()` in `src/lib/utils/securityHeaders.ts`. These include CSP, HSTS, X-Frame-Options, Referrer-Policy, etc.

Do not weaken these headers in new routes.

## Audit logging

Every mutation that affects user data should call `logActivity`. It is fire-and-forget (never awaited).

```ts
import { logActivity } from '@/lib/services/auditLog';

logActivity({
  userId: auth.userId,
  action: 'chore.complete',
  entityType: 'chore',
  entityId: choreId,
  summary: `${userName} completed "${choreTitle}"`,
});
```
