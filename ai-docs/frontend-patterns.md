# Frontend Patterns

## Pages

Each feature page has the same structure:

```
src/app/chores/
  page.tsx           # Thin server component — sets metadata, renders the view
  ChoresView.tsx     # 'use client' — full UI
  useChoresViewData.ts  # Data-fetching hook (fetch + state)
  useChoreModals.ts     # Modal open/close/submit logic
  ChoreItem.tsx
  ChoreModal.tsx
  ChoreGroupCard.tsx
  ...
```

### page.tsx pattern

```ts
// src/app/chores/page.tsx
import { ChoresView } from './ChoresView';

export const metadata = {
  title: 'Chores',
  description: 'Family chore tracker',
};

export default function ChoresPage() {
  return <ChoresView />;
}
```

No data fetching in the page file itself. Server components for metadata only.

### View component pattern

```tsx
'use client';

export function ChoresView() {
  const { chores, loading, error, refreshChores } = useChoresViewData();
  const { saveNewChore, ... } = useChoreModals({ refreshChores, ... });

  if (loading) return <LoadingSpinner />;
  if (error) return <ErrorMessage error={error} />;

  return (
    <PageWrapper>
      <SubpageHeader title="Chores" />
      {/* feature UI */}
    </PageWrapper>
  );
}
```

### Data hook pattern

Hooks in `use*ViewData.ts` call the project's REST API via `fetch()`.
They manage:
- loading / error state
- the fetched data
- filter/sort state derived from the data
- inline mutations (complete, delete, etc.)

```ts
export function useChoresViewData() {
  const [chores, setChores] = useState<Chore[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  const fetchChores = useCallback(async () => {
    setLoading(true);
    try {
      const res = await fetch('/api/chores');
      if (!res.ok) throw new Error('Failed to fetch');
      setChores(await res.json());
    } catch (e) {
      setError(e instanceof Error ? e.message : 'Unknown error');
    } finally {
      setLoading(false);
    }
  }, []);

  useEffect(() => { fetchChores(); }, [fetchChores]);

  return { chores, loading, error, refreshChores: fetchChores };
}
```

## Layout primitives

Import from `@/components/layout`:

| Component | Purpose |
|---|---|
| `<PageWrapper>` | Full-height page shell with background |
| `<SubpageHeader title="…">` | Standard page title bar with back button |
| `<FilterBar>` | Horizontal filter/sort strip |
| `<SortSelect>` | Dropdown for sort order |
| `<FilterDropdown>` | Multi-select filter chip |
| `<PersonFilter>` | Family member avatar filter row |

## UI components

All in `src/components/ui/` — Radix UI primitives + custom wrappers with Tailwind.

Common primitives:

```tsx
import { Button } from '@/components/ui/button';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Dialog, DialogContent, DialogHeader, DialogTitle, DialogTrigger } from '@/components/ui/dialog';
import { Input } from '@/components/ui/input';
import { Label } from '@/components/ui/label';
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from '@/components/ui/select';
import { Switch } from '@/components/ui/switch';
import { Badge } from '@/components/ui/badge';
import { Tabs, TabsContent, TabsList, TabsTrigger } from '@/components/ui/tabs';
import { Toast } from '@/components/ui/toast';
```

## Auth context

The `useAuth()` hook provides the logged-in user and a `requireAuth()` client-side guard:

```tsx
import { useAuth } from '@/components/providers';

const { user, requireAuth } = useAuth();

// Guard a mutation (prompts PIN pad if needed)
const handleDelete = () => {
  if (!requireAuth('parent')) return;
  // proceed
};
```

`user` shape: `{ id, name, role, color, avatarUrl }`.

## Styling conventions

- Tailwind utility classes throughout — no CSS modules or styled-components.
- Use `cn()` from `@/lib/utils` for conditional class merging:
  ```ts
  import { cn } from '@/lib/utils';
  className={cn('base-classes', isActive && 'active-classes')}
  ```
- Dark mode is toggled by a class on the root element; use `dark:` variants.
- Colors use CSS variables defined in `tailwind.config.js` (e.g. `bg-background`, `text-foreground`, `bg-card`, `text-muted-foreground`).

## Icons

Use `lucide-react`. Import only what you need:

```tsx
import { Plus, Trash2, CheckCircle } from 'lucide-react';
```

## Forms

Use `react-hook-form` + Zod for complex forms, or plain controlled state for simple inputs.

## Mobile / responsive

The app targets wall displays and tablets first, phones second.
Use the `useIsMobile()` hook from `@/lib/hooks/useIsMobile` to branch logic when needed.
Most feature views render differently on mobile (simplified layout, reduced columns).

## Drag and drop

`@dnd-kit` is used for sortable lists (shopping categories, task reordering).
See `src/app/shopping/useShoppingDragReorder.ts` for a reference implementation.

## Toasts / notifications

Use the `useToast()` hook from `@/components/ui/use-toast`:

```ts
const { toast } = useToast();
toast({ title: 'Chore completed!', variant: 'default' });
toast({ title: 'Error', description: err.message, variant: 'destructive' });
```

## Settings sections

Adding a setting section:
1. Create `src/app/settings/sections/MySection.tsx`
2. Import and render it in `src/app/settings/SettingsView.tsx` under a new entry in the `sections` array and the conditional render block at the bottom.
