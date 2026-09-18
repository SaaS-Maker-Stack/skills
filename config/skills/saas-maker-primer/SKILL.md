---
name: saas-maker-primer
description: "INVOKE FIRST for any work on a project built with SaaS Maker (FastAPI + SQLModel + PostgreSQL backend, React 19 + shadcn/ui frontend, neutral admin panel, Kamal deploys) — extending it, fixing it, restyling it or operating it. Establishes the three services, the multi-tenant isolation rule, the quality gates, how to apply an external DESIGN.md (brand color, radius, fonts) without breaking the system, and routes you to the right skill (cli, add-module, deploy) for the task at hand."
---

# SaaS Maker — primer & router

SaaS Maker is a **multi-tenant SaaS starter**: a project generated with
`saas-maker new` is three services that share nothing but an API contract, with
authentication, organizations, members, invitations, email verification, sessions
and a platform admin panel already working. This skill orients you and points you to
the next skill. It is a router, not a manual — the project itself carries its docs.

## The one rule: read the project's own docs, don't guess

Every generated project ships them; read before writing code:

- `CLAUDE.md` (root) — layout, ports, quick start, the module list.
- `backend/CLAUDE.md` — architecture, auth flow, dependencies, endpoints, tests.
- `frontend/CLAUDE.md` — stack, directory structure, React Query keys, routes.
- `DESIGN.md` (root) — the design system. **Read it before touching any UI.**

Template source (for what a pristine file looks like):
`https://raw.githubusercontent.com/SaaS-Maker-Stack/saas-maker-<backend|frontend|admin>/main/<path>`.

## The three services

| Dir | Stack | Port | Role |
|-----|-------|------|------|
| `backend/` | Python 3.14, FastAPI, SQLModel, Alembic, PostgreSQL (asyncpg), PyJWT, uv, ruff | 8090 | The API. Tenant endpoints + `/admin/*` endpoints |
| `frontend/` | React 19, TypeScript, Vite, React Router 8, TanStack Query, shadcn/ui, Tailwind 4, npm | 5190 | The tenant app (what customers use) |
| `admin/` | Same stack as frontend, gray palette on purpose | 5191 | Platform admin panel (separate `admin_users` auth) |

Each directory is **its own git repository** and deploys independently with Kamal.
There is no root repo: commit inside each service. Local infra is native (Postgres +
`make dev` + `npm run dev`); `docker-compose.yml` at the root is an optional
Postgres + Mailpit for people without them.

## Non-negotiables

1. **Tenant isolation.** Every tenant table has `organization_id`. Every service function
   takes `organization_id` and filters by it. A row of another organization answers
   **404, never 403** (don't reveal existence). The org comes from the JWT
   (`CurrentUser.organization_id`), never from the request body or query string.
2. **Roles.** owner > admin > member. Read = any member; write = `require_role("admin")`
   unless the feature says otherwise. Admin-panel endpoints use `CurrentAdmin` /
   `require_admin`, a different auth realm — never mix the two.
3. **Layers.** `controllers/` (HTTP only) → `services/` (business logic, all queries) →
   `models/` (SQLModel). No queries in controllers. Schemas in `schemas/`.
4. **Migrations are hand-reviewed.** `make makemigrations m="..."` then read the file
   (Alembic autogenerate misses indexes and server defaults); one head at a time.
5. **Frontend server state lives in React Query** (`hooks/use<Thing>.ts`, keys
   `['things', params]`, mutations invalidate). Components in
   `components/features/<thing>/`; pages only compose. Every list has the four states:
   loading, error, empty, data.
6. **Quality gates before you say "done"** — run them, don't assume:
   ```bash
   cd backend  && make lint && make test          # ruff + format check; pytest needs local Postgres
   cd frontend && npm run lint && npm run typecheck && npm test
   cd admin    && npm run lint && npm run typecheck && npm test
   ```
   CI runs the same per repo on push/PR. New endpoint ⇒ test in `backend/tests/`
   (use `client`, `register()`, `auth_headers()` from `conftest.py`, and always add the
   cross-organization 404 case). New page/component ⇒ `*.test.tsx` next to it using
   `src/test/render.tsx`.
7. **Code and comments in English; UI copy in Spanish** (tone rules in `DESIGN.md`).
8. **Secrets** live in `.env` (dev) and `.kamal/secrets` (prod); both are gitignored.
   Never print or commit them. `.env.example` is the documented shape.

## Design system in one paragraph

`DESIGN.md` is neutral, Linear-inspired, light-first: near-white canvas, gray surfaces
separated by hairline borders (no shadows), Inter at 400/500/600, and **one chromatic
accent** derived from a single variable, `--brand-hue`, in `frontend/src/index.css`.
`--primary`, hover, ring, `--sidebar-primary` and `--chart-1..5` (light and dark) are all
`oklch(L C var(--brand-hue))`; changing the hue re-brands the whole app. The admin panel
uses the same system with the accent removed (near-black primary) and **must stay gray**.
Never hard-code a color in a component; use the tokens. `--radius: 0.625rem`.

### Applying an external DESIGN.md (e.g. from getdesign.md / awesome-design-md)

Devs sometimes bring a `DESIGN.md` of a reference product. Do **not** replace the
project's `DESIGN.md` wholesale or copy its full palette: the components depend on the
token names and the neutral structure. Extract only what the customization section of the
project's `DESIGN.md` allows, in this order:

1. **Brand color** → the reference's primary/accent hex → OKLCH hue → `--brand-hue`.
   Any converter works (or `python -c` with a quick sRGB→OKLab→atan2). Sanity: blue
   ≈250, indigo ≈265, violet ≈300, rose ≈350, orange ≈25, green ≈145, teal ≈185. Keep
   chroma/lightness as defined; if the reference is a gray/black brand, leave the hue and
   note it (the admin is already the neutral version).
2. **Radius** → `--radius` (0.5rem sharper, 0.75rem softer), same value in `admin/`.
3. **Fonts** → only if the brand demands it: swap the Google Fonts `<link>` in
   `index.html` and `--font-sans` in `index.css` of **both** apps; keep the weights 400/500/600.
4. **Logo** → replace the `Building2` icon in `AuthLayout`, `Sidebar` and the admin
   `LoginPage` with the brand mark at 24px.
5. Record the change in the project's `DESIGN.md` (name, `primary` token values, fonts).

Then check contrast: the primary at 0.55 lightness meets AA on white for text ≥14px/500,
but yellow/green hues need a darker lightness for text use (see "Known Gaps"). Everything
else in the reference (spacing scale, shadows, secondary palettes, marketing styles) is
**not** imported.

## Which skill next?

Evaluate in order, stop at the first match:

1. **Creating a new project, or using the generator** (`saas-maker new`,
   `saas-maker generate module`) → load **`saas-maker-cli`**.
2. **Adding a tenant-scoped resource** (a table + API + page), by generator or by hand,
   or anything that is "CRUD-shaped" → load **`saas-maker-add-module`**.
3. **Deploying or operating on a server** (Kamal: first install, ship, logs, rollback,
   migrations in prod, first admin user) → load **`saas-maker-deploy`**.
4. **Anything else** (auth changes, admin-panel features, emails, restyling) → work
   from the `CLAUDE.md` files and `DESIGN.md`, keeping the non-negotiables above.

## Anti-patterns

- ❌ Querying without `organization_id`, or taking it from the request.
- ❌ Returning 403 for another org's row.
- ❌ Tinting the admin panel or hard-coding colors.
- ❌ Importing a whole external palette instead of the hue + radius + fonts.
- ❌ Declaring done without the lint/typecheck/test gates.
- ❌ Committing `.env`, `.kamal/secrets`, or pasting their values into chat.
