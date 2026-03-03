# Procurement Spec Workspace

Centralized specification workspace for the AKDITAL Procurement system (APP).

## Layout

- `specs/` — all feature specifications (API-only, frontend-only, or cross-cutting)
- `.specify/` — Specify tooling (scripts, templates, memory, constitution)
- `procurement-api/` → symlink to the backend project
- `procurement-app/` → symlink to the frontend project

## Spec Targeting

Every spec declares a **Target** field: `api`, `app`, or `both`.

When working on a spec, load the relevant project context:

- **Target: api** → read `procurement-api/CLAUDE.md` for backend guidelines
- **Target: app** → read `procurement-app/CLAUDE.md` → `procurement-app/AGENTS.md` for frontend guidelines
- **Target: both** → read both

### Project References

- **Backend**: [procurement-api/CLAUDE.md](procurement-api/CLAUDE.md)
  - PHP 8.2 / Symfony 7.x / API Platform 4.x / PostgreSQL
  - Docker (FrankenPHP), tests via `docker compose exec php php bin/phpunit`

- **Frontend**: [procurement-app/AGENTS.md](procurement-app/AGENTS.md)
  - React 19 / TypeScript / Vite / pnpm
  - shadcn/ui, Zustand, React Query v3, React Router v7
  - Tests via `pnpm test`

## Branching

- Feature branches target `dev`, never `main`
- Branch naming: `NNN-short-name` (sequential number + kebab-case)
- Tasks in specs MUST tag their target: `[API]`, `[APP]`, or `[BOTH]`
