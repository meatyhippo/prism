# Project Structure

## Root

```
prism/
├── src/                  # All application source
├── drizzle/              # Migration SQL files (generated)
├── e2e/                  # Playwright end-to-end tests
├── public/               # Static assets, PWA files
├── docker/               # Nginx config, entrypoint
├── scripts/              # Bash helpers (install, backup, deploy)
├── alexa/                # Alexa skill definition
├── docs/                 # User-facing MkDocs site
├── ai-docs/              # AI developer reference (this folder)
├── docker-compose.yml    # Production compose
├── docker-compose.dev.yml
├── next.config.js
├── drizzle.config.ts
├── tailwind.config.js
├── tsconfig.json
└── .env / .env.example
```

## src/app — Pages and API Routes

Every folder under `src/app/` maps to a URL segment (Next.js App Router).

```
src/app/
├── page.tsx                    # / → Dashboard
├── DashboardClient.tsx         # Client shell; loads widgets
├── layout.tsx                  # Root HTML layout
├── calendar/                   # /calendar
├── chores/                     # /chores
├── goals/                      # /goals
├── messages/                   # /messages
├── shopping/                   # /shopping
├── tasks/                      # /tasks
├── travel/                     # /travel
├── weekend/                    # /weekend
├── help/                       # /help
├── settings/                   # /settings
│   └── sections/               # One component per settings section
└── api/                        # All API route handlers
    ├── auth/                   # login, logout, OAuth callbacks
    ├── calendars/              # CRUD + sync
    ├── chores/                 # CRUD + complete + approve
    ├── events/
    ├── goals/
    ├── meals/
    ├── messages/
    ├── photos/
    ├── recipes/
    ├── settings/
    ├── shopping-items/
    ├── shopping-lists/
    ├── tasks/ + task-lists/ + task-sources/
    ├── travel/
    └── v1/voice/               # Voice API (Alexa)
```

## src/lib — Shared Server Library

```
src/lib/
├── api/
│   ├── withAuth.ts             # Auth wrapper for route handlers
│   ├── voiceResponse.ts        # Voice API helpers
│   └── voicePhrases.ts
├── auth/
│   ├── session.ts              # Redis session CRUD + rate limit
│   ├── requireAuth.ts          # requireAuth / requireRole / optionalAuth
│   ├── apiTokens.ts            # Machine-to-machine bearer tokens
│   └── settingsAuth.ts         # Extra PIN gate for settings
├── cache/
│   └── getRedisClient.ts       # Singleton Redis connection with retry
├── db/
│   ├── client.ts               # Drizzle + postgres.js singleton
│   ├── schema.ts               # ALL table definitions
│   ├── seed.ts                 # Dev seed data
│   └── clear.ts
├── integrations/
│   ├── credentialStore.ts      # DB-first credential lookup with env fallback
│   ├── google-calendar.ts
│   ├── onedrive.ts
│   ├── kroger/                 # client.ts + redirect.ts + tokens.ts
│   ├── tasks/                  # TaskProvider interface + MS-Todo + Google Tasks
│   ├── shopping/
│   ├── wish-items/
│   ├── weather.ts              # Provider switcher
│   ├── openmeteo.ts
│   ├── openweather.ts
│   ├── pirateweather.ts
│   ├── immich.ts
│   └── gmail.ts
├── services/
│   ├── auditLog.ts             # Fire-and-forget logActivity()
│   ├── calendar-sync.ts        # Google + iCal sync logic
│   ├── photo-cache.ts
│   ├── photo-storage.ts
│   └── ...
├── server/
│   └── calendarSyncCron.ts     # Startup cron (started in instrumentation.ts)
├── utils/
│   ├── crypto.ts               # AES-256-GCM encrypt / decrypt
│   ├── safeFetch.ts            # URL allow-list validation
│   ├── logError.ts
│   └── securityHeaders.ts
└── validations/
    └── index.ts                # Zod schemas for all API inputs
```

## src/components — UI Components

```
src/components/
├── ui/                         # Radix UI primitives + project components
│   ├── button.tsx
│   ├── dialog.tsx
│   ├── card.tsx
│   └── ...
├── layout/                     # PageWrapper, SubpageHeader, FilterBar, etc.
├── dashboard/                  # Widget components used by DashboardClient
├── calendar/                   # Reusable calendar renderers
├── widgets/                    # Individual dashboard widgets
├── providers/                  # React context providers (auth, theme, etc.)
├── auth/                       # PinPad login UI
├── away-mode/
├── babysitter-mode/
├── screensaver/
└── modals/                     # Shared modals (confirm dialog, etc.)
```

## Path alias

All imports use `@/` which resolves to `src/`. Configured in `tsconfig.json`.

```ts
import { db } from '@/lib/db/client';
import { Button } from '@/components/ui/button';
```
