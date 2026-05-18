# Data Layer — Database and ORM

## Stack

- **PostgreSQL 15** (Docker container `prism-db`)
- **Drizzle ORM** (`drizzle-orm`) — type-safe query builder
- **postgres.js** — low-level PostgreSQL driver with connection pooling
- **Drizzle Kit** — schema diffing and migration generation

## DB client

`src/lib/db/client.ts` exports a lazy-initialised Drizzle proxy:

```ts
import { db } from '@/lib/db/client';
```

The connection pool is created on first use (not at build time). Max 10 connections, 30 s idle timeout.

Never create a second db instance. Always import from `@/lib/db/client`.

## Schema

All table definitions live in **one file**: `src/lib/db/schema.ts`.

Current tables (as of v1.8):

| Table | Purpose |
|---|---|
| `users` | Family members (parent / child / guest) with PIN hash, color, avatar |
| `calendarGroups` | Logical grouping of calendars for display |
| `calendarSources` | External calendar connections (Google, iCal, etc.) with encrypted OAuth tokens |
| `events` | Calendar events, synced or locally created |
| `calendarNotes` | Day-level freeform notes |
| `taskLists` | Named task list containers |
| `taskSources` | External task provider connections (MS To-Do, Google Tasks) |
| `tasks` | Individual tasks with assignment, due date, priority, external sync fields |
| `chores` | Recurring household chores with frequency and point values |
| `choreCompletions` | Completion log with approval and photo evidence |
| `shoppingLists` | Shopping list containers |
| `shoppingItems` | Line items with category, quantity, external IDs |
| `meals` | Meal plan entries |
| `recipes` | Stored recipe cards with ingredients and steps |
| `goals` | Goals linked to chore point accrual |
| `messages` | Family message board posts |
| `travelPins` | Geo-pins for the travel map |
| `travelTrips` | Multi-stop trip routes |
| `weekendIdeas` | Weekend outing suggestions |
| `photos` | Photo metadata (OneDrive / local / Immich sources) |
| `settings` | Key-value store for app settings and encrypted credentials |
| `auditLogs` | Activity log for all mutations |
| `maintenanceItems` | Home maintenance tasks |
| `apiTokens` | Machine-to-machine bearer token records |
| `babysitterInfo` | Caregiver-facing house info blocks |
| `busRoutes` | School bus tracking configuration |

## Querying

Drizzle uses a builder API that maps directly to SQL:

```ts
import { db } from '@/lib/db/client';
import { chores, users } from '@/lib/db/schema';
import { eq, and, gte } from 'drizzle-orm';

// Select
const result = await db
  .select()
  .from(chores)
  .where(and(eq(chores.assignedTo, userId), eq(chores.enabled, true)));

// Insert
const [inserted] = await db
  .insert(chores)
  .values({ title, category, frequency, ... })
  .returning();

// Update
await db
  .update(chores)
  .set({ lastCompleted: new Date() })
  .where(eq(chores.id, choreId));

// Delete
await db.delete(chores).where(eq(chores.id, choreId));

// Relations / joins (relational query API)
const user = await db.query.users.findFirst({
  where: eq(users.id, userId),
  with: { choreCompletions: true },
});
```

## Migrations

1. Edit `src/lib/db/schema.ts`
2. Run `npm run db:generate` — produces a new SQL file in `drizzle/`
3. Commit the generated SQL file alongside the schema change
4. Migrations run automatically on container start via `scripts/migrate.js`

**Never hand-edit the generated SQL files.**  
**Never run `db:push` in production** — it bypasses migration history.

## UUIDs and timestamps

Every table uses:
- `id: uuid().defaultRandom().primaryKey()`
- `createdAt: timestamp().defaultNow().notNull()`
- `updatedAt: timestamp().defaultNow().notNull()`

Set `updatedAt` manually on updates — Drizzle does not auto-update it:

```ts
await db.update(chores).set({ title: newTitle, updatedAt: new Date() }).where(...);
```

## JSON columns

Several tables use `jsonb` columns (`preferences`, `metadata`, `syncErrors`, etc.).
Use TypeScript casting or Zod to validate the shape before reading:

```ts
const prefs = user.preferences as { theme?: string } ?? {};
```

## Indexes

Indexes are declared inline in the schema's table options object:

```ts
}, (table) => ({
  assignedToIdx: index('chores_assigned_to_idx').on(table.assignedTo),
}));
```

Add an index whenever you add a column that will be used in a `WHERE` clause on hot paths.
