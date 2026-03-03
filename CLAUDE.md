# Procurement Spec Workspace

Centralized specification workspace for the AKDITAL Procurement system (APP).
This workspace is the orchestration layer for the development pipeline — from Jira task to shipped PR.

## Layout

- `specs/` — all feature specifications (API-only, frontend-only, or cross-cutting)
- `.specify/` — Specify tooling (scripts, templates, memory, constitution)
- `procurement-api/` → symlink to the backend project
- `procurement-app/` → symlink to the frontend project

---

## Development Pipeline

### Step 1 — Assess the task

Before anything else, fetch the Jira issue and assess it:

| Question | Action |
|----------|--------|
| Is it a duplicate? | Close it in Jira, link to the original. Stop. |
| Is it invalid or incompatible with the architecture? | Add a comment in Jira explaining why. Flag to PO. Stop. |
| Is it valid? | Continue to Step 2. |

### Step 2 — Classify and pick a track

| Type | Examples | Track |
|------|----------|-------|
| **Bug** | UI glitch, wrong label, minor fix | → **Direct** |
| **Simple task** | Translation, small config, one-file change | → **Direct** |
| **User story / Feature** | New workflow, new page, new API endpoint | → **Spec** |
| **Complex / cross-project** | Spans API + frontend, architectural change | → **Full pipeline** |

### Track: Direct
1. Implement the fix
2. Open PR (title includes Jira key: `ACHAT-XXX: ...`)
3. Post PR link as Jira comment

### Track: Spec
1. Create spec in `specs/` from the Jira task (pull description as input, flesh out edge cases and acceptance criteria)
2. Implement from the spec
3. Open PR with spec-derived description
4. Post PR link as Jira comment

### Track: Full pipeline
1. Create spec → review with PO (push key sections back to Jira for validation)
2. Once agreed: generate plan + tasks
3. Implement task by task
4. Open PR per logical unit
5. Update Jira on completion

---

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

---

## Branching & PRs

- Feature branches target `dev`, never `main`
- Branch naming: `ACHAT-XXX-short-name` (Jira key + kebab-case description) — this is the traceability link
- PR title must include the Jira key: `ACHAT-XXX: short description` — Jira auto-links it
- PR description maps to spec sections: what changed, why, how to test
- One branch per Jira issue, even for simple fixes
- Tasks in specs MUST tag their target: `[API]`, `[APP]`, or `[BOTH]`

## Definition of Ready

Before starting any task, the Jira issue must have:
- A clear description of what is needed (or what is broken)
- Steps to reproduce for bugs

If not, post a comment in Jira asking for clarification. Do not write code against an unclear task.

## Bugs

Do not create a formal spec for bugs. Investigation notes in a short comment are enough.
Reserve the spec format for tasks with real design decisions.
