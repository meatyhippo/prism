# Adding a New Feature Module

This is the step-by-step checklist for adding a brand new feature to Prism (e.g. a new "Announcements" board or a "Habit Tracker" page).

---

## 1. Plan the data model

Decide what columns the new table needs and how it relates to `users`.
Add the table to `src/lib/db/schema.ts`:

```ts
export const announcements = pgTable('announcements', {
  id: uuid('id').defaultRandom().primaryKey(),
  title: varchar('title', { length: 255 }).notNull(),
  body: text('body').notNull(),
  postedBy: uuid('posted_by').references(() => users.id, { onDelete: 'set null' }),
  createdAt: timestamp('created_at').defaultNow().notNull(),
  updatedAt: timestamp('updated_at').defaultNow().notNull(),
}, (table) => ({
  postedByIdx: index('announcements_posted_by_idx').on(table.postedBy),
}));
```

Then generate the migration:

```bash
npm run db:generate
```

Commit both the schema change and the generated SQL file in `drizzle/`.

---

## 2. Add Zod validation schemas

Add create/update schemas to `src/lib/validations/index.ts`:

```ts
export const createAnnouncementSchema = z.object({
  title: z.string().min(1).max(255),
  body: z.string().min(1).max(10000),
  postedBy: uuidSchema.optional(),
});

export const updateAnnouncementSchema = createAnnouncementSchema.partial();
```

---

## 3. Create the API routes

```
src/app/api/announcements/
  route.ts          # GET (list), POST (create)
  [id]/
    route.ts        # GET (single), PUT (update), DELETE
```

Follow the patterns in [api-routes.md](api-routes.md):
- Wrap mutations with `withAuth` and the appropriate `permission`
- Validate all bodies with the Zod schema
- Call `logActivity` on every mutation
- Return consistent response shapes

---

## 4. Create the page

```
src/app/announcements/
  page.tsx              # Metadata + render view
  AnnouncementsView.tsx # 'use client' — full UI
  useAnnouncementsData.ts  # Data-fetch + state hook
  AnnouncementCard.tsx  # Card component
  AnnouncementModal.tsx # Create/edit modal
```

### page.tsx

```ts
import { AnnouncementsView } from './AnnouncementsView';

export const metadata = { title: 'Announcements' };
export default function AnnouncementsPage() {
  return <AnnouncementsView />;
}
```

### AnnouncementsView.tsx

```tsx
'use client';
import { PageWrapper, SubpageHeader } from '@/components/layout';
import { useAnnouncementsData } from './useAnnouncementsData';

export function AnnouncementsView() {
  const { announcements, loading, error } = useAnnouncementsData();
  // render...
}
```

### useAnnouncementsData.ts

```ts
'use client';
import { useState, useEffect, useCallback } from 'react';

export function useAnnouncementsData() {
  const [announcements, setAnnouncements] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  const fetch_ = useCallback(async () => {
    setLoading(true);
    try {
      const res = await fetch('/api/announcements');
      if (!res.ok) throw new Error('Failed');
      setAnnouncements(await res.json());
    } catch (e) {
      setError(e instanceof Error ? e.message : 'Unknown');
    } finally {
      setLoading(false);
    }
  }, []);

  useEffect(() => { fetch_(); }, [fetch_]);
  return { announcements, loading, error, refresh: fetch_ };
}
```

---

## 5. Add navigation

Navigation lives in two places:

1. **Mobile / desktop nav bar** — `src/components/layout/Nav.tsx` (or equivalent). Add a `<NavItem>` entry.
2. **Dashboard** — if the feature should appear as a widget, add a widget component under `src/components/widgets/` and register it in `src/app/DashboardClient.tsx`.

---

## 6. Write unit tests

Create `src/app/api/announcements/__tests__/announcements.test.ts`.
See [testing.md](testing.md) for patterns.

---

## 7. Write E2E tests (optional but encouraged)

Create `e2e/announcements.spec.ts`.
See [testing.md](testing.md) for patterns.

---

## Checklist

- [ ] Schema added + migration generated and committed
- [ ] Zod schemas in `src/lib/validations/index.ts`
- [ ] API routes with auth, validation, audit logging
- [ ] Page + view component + data hook
- [ ] Navigation entry added
- [ ] Unit tests for API routes
