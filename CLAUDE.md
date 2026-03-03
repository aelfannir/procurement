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

Fetch the Jira issue, then:

**If the task is back In Progress (QA rejection):**
1. Find the linked QA test issue
2. Read the rejection reason and comments before touching any code
3. Fix based on the QA feedback, not assumptions

**If starting fresh:**
1. Search for related open issues: `project = ACHAT AND statusCategory != Done AND text ~ "keyword"` — catch duplicates or overlapping bugs
2. If a duplicate is found: close this issue, link to the original. Stop.
3. If invalid or incompatible: comment in Jira explaining why. Flag to PO. Stop.
4. If the description is vague or incomplete: update it in Jira before starting
5. Assign to user if unassigned
6. Transition to In Progress (transition_id: `2`)
7. Continue to Step 2

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
3. Post Jira comment with what changed + test steps for QA

### Track: Spec
1. Create spec in `specs/` from the Jira task (pull description as input, flesh out edge cases and acceptance criteria)
2. Implement from the spec
3. Open PR with spec-derived description
4. Post Jira comment with what changed + test steps for QA

### Track: Full pipeline
1. Create spec → review with PO (push key sections back to Jira for validation)
2. Once agreed: generate plan + tasks
3. Implement task by task
4. Open PR per logical unit
5. Post Jira comment with what changed + test steps for QA

### Jira QA comment format
Post this after every PR is opened — QA has no GitHub access:

```
✅ Implemented — pending staging deployment

*What changed:*
[1-2 sentences describing the change in plain language]

*How to test:*
1. [Step 1]
2. [Step 2]
3. Expected result: [expected result]
```

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
