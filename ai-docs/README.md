# Prism — AI Developer Reference

This folder contains concise, accurate reference docs for an AI assistant working on the Prism codebase.
Each document focuses on one concern. Read only what you need for the task at hand.

| Document | When to read it |
|---|---|
| [project-structure.md](project-structure.md) | Navigating the repo, understanding folder conventions |
| [api-routes.md](api-routes.md) | Adding or editing a Next.js API route handler |
| [data-layer.md](data-layer.md) | Database client, Drizzle ORM, schema, migrations |
| [auth-security.md](auth-security.md) | Sessions, roles, CSRF, encryption, API tokens |
| [frontend-patterns.md](frontend-patterns.md) | Pages, view components, hooks, layout primitives |
| [new-module.md](new-module.md) | Step-by-step: add a brand new feature module |
| [new-integration.md](new-integration.md) | Step-by-step: add a new external service integration |
| [testing.md](testing.md) | Unit tests (Jest), E2E tests (Playwright) |

## Quick facts

- **Stack**: Next.js 15 App Router · React 18 · TypeScript · Drizzle ORM · PostgreSQL · Redis
- **Entry point**: `src/app/page.tsx` → `src/app/DashboardClient.tsx`
- **API base path**: `src/app/api/`
- **Shared library**: `src/lib/` (auth, db, integrations, services, utils, validations)
- **UI component library**: `src/components/` (Radix UI + Tailwind, wrapped in `src/components/ui/`)
- **Deployment**: Docker Compose — `docker-compose.yml`
- **Database migrations**: `drizzle/` SQL files, generated with `npm run db:generate`
- **Environment secrets**: `.env` file (see `.env.example`). Encrypted at rest with `ENCRYPTION_KEY` (AES-256-GCM).
