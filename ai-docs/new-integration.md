# Adding a New External Service Integration

Prism integrations follow a consistent pattern. This document walks through adding a new one end-to-end, using a hypothetical "Todoist" task provider as the running example.

---

## Two categories of integration

| Type | Examples | Pattern |
|---|---|---|
| **Task/data provider** (bidirectional sync) | MS To-Do, Google Tasks | `TaskProvider` interface + registry |
| **Data source** (read-only or one-way push) | Google Calendar, Kroger, weather, photos | Service module + credential store |

---

## Category 1: Task-like provider (syncing records)

### 1a. Implement the `TaskProvider` interface

Create `src/lib/integrations/tasks/todoist.ts`:

```ts
import type { TaskProvider, TaskProviderTokens, ExternalTaskList, ExternalTask, SyncResult } from './types';

export const todoistProvider: TaskProvider = {
  providerId: 'todoist',
  displayName: 'Todoist',

  async fetchLists(tokens: TaskProviderTokens): Promise<ExternalTaskList[]> {
    const res = await fetch('https://api.todoist.com/rest/v2/projects', {
      headers: { Authorization: `Bearer ${tokens.accessToken}` },
    });
    if (!res.ok) throw new Error(`Todoist API error: ${res.status}`);
    const projects = await res.json();
    return projects.map((p: { id: string; name: string }) => ({ id: p.id, name: p.name }));
  },

  async fetchTasks(tokens, listId) { /* ... */ },
  async createTask(tokens, input) { /* ... */ },
  async updateTask(tokens, taskId, input) { /* ... */ },
  async deleteTask(tokens, taskId) { /* ... */ },
  async syncBidirectional(tokens, listId, localTasks) { /* ... */ },
};
```

### 1b. Register the provider

In `src/lib/integrations/tasks/index.ts`, add to the registry:

```ts
import { todoistProvider } from './todoist';

export const taskProviders: Record<string, TaskProvider> = {
  microsoft_todo: microsoftTodoProvider,
  google_tasks: googleTasksProvider,
  todoist: todoistProvider,              // ← add here
};
```

### 1c. Add credentials to the credential store

In `src/lib/integrations/credentialStore.ts`, add a getter function following the existing pattern:

```ts
export async function getTodoistCredentials(): Promise<{ apiKey: string } | null> {
  const stored = await getSetting('credentials.todoist');
  if (stored?.apiKey) return { apiKey: safeDecrypt(stored.apiKey) ?? stored.apiKey };
  const apiKey = process.env.TODOIST_API_KEY;
  if (!apiKey) return null;
  return { apiKey };
}
```

---

## Category 2: Data source (calendar / weather / photos)

### 2a. Create the service module

Create `src/lib/integrations/myservice.ts`:

```ts
/**
 * MyService integration
 */

export interface MyServiceData {
  // define the shape of what the service returns
}

export async function fetchMyServiceData(params: { apiKey: string }): Promise<MyServiceData> {
  const res = await fetch(`https://api.myservice.com/data?key=${params.apiKey}`);
  if (!res.ok) throw new Error(`MyService API error: ${res.status}`);
  return res.json();
}
```

**Important**: never inline credentials. Use `credentialStore.ts`.

### 2b. Add credentials

Add a getter in `src/lib/integrations/credentialStore.ts` (same pattern as above).

---

## OAuth flows

If the service uses OAuth 2.0:

### OAuth route files

```
src/app/api/auth/myservice/
  route.ts          # GET — builds the authorization URL, redirects the user
  callback/
    route.ts        # GET — exchanges code for tokens, stores them encrypted
```

Reference: `src/app/api/auth/google/route.ts` and `src/app/api/auth/google/callback/route.ts`.

### Token storage

Tokens go into the relevant table (e.g. `calendarSources.accessToken`, `taskSources.accessToken`) as AES-256-GCM encrypted strings:

```ts
import { encrypt, decrypt } from '@/lib/utils/crypto';

// Store
await db.update(taskSources).set({
  accessToken: encrypt(tokens.access_token),
  refreshToken: encrypt(tokens.refresh_token),
  tokenExpiresAt: new Date(Date.now() + tokens.expires_in * 1000),
});

// Use
const raw = decrypt(source.accessToken!);
```

### Token refresh

Check for expiry before every API call:

```ts
function tokenNeedsRefresh(expiresAt: Date | null): boolean {
  if (!expiresAt) return true;
  return expiresAt <= new Date(Date.now() + 5 * 60 * 1000); // 5-min buffer
}
```

If refresh fails with a `token_revoked` / 401 error, mark the source as needing re-auth and surface a notification in the UI.

---

## Settings UI

### Add a connection card

Create `src/app/settings/sections/MyServiceConnectionCard.tsx` — a card that shows connection status and a "Connect" / "Disconnect" button.

Register it in the relevant settings section (e.g. `ConnectedAccountsSection.tsx`).

### Feature toggle (optional)

If the integration is opt-in, add a toggle to `src/app/settings/sections/FeaturesSection.tsx` and gate the integration behind a feature flag check using the `settings` table key `features.myservice`.

---

## Background sync (optional)

If the integration requires periodic background sync:

1. Add a sync function to `src/lib/services/` (e.g. `myservice-sync.ts`)
2. Register it in `src/lib/server/calendarSyncCron.ts` alongside the existing calendar sync schedule

---

## Checklist

- [ ] Service module in `src/lib/integrations/myservice.ts`
- [ ] Credentials getter in `src/lib/integrations/credentialStore.ts`
- [ ] OAuth routes (if needed): `src/app/api/auth/myservice/route.ts` + `callback/route.ts`
- [ ] Tokens stored encrypted; decrypt before use
- [ ] Token refresh logic
- [ ] Settings connection card in the appropriate settings section
- [ ] Provider registered in provider registry (if task-like)
- [ ] Background sync wired into cron (if periodic)
- [ ] Docs updated in `docs/features/`
