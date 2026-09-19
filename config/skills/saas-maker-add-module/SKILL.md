---
name: saas-maker-add-module
description: "INVOKE when adding a tenant-scoped resource (a CRUD module: table + API + page) to a project built from SaaS Maker — FastAPI + SQLModel backend, React 19 + shadcn frontend, multi-tenant by organization. Covers the backend checklist (model, schemas, service, controller, migration, tests), the frontend checklist (types, hook, feature components with the four UI states, page, route, sidebar, test), the `generator:` anchors, and the non-negotiable isolation rule (every query filters by organization_id; other organizations get 404). Copies the `projects` reference module."
---

# Add a module (tenant-scoped resource)

A **module** in a SaaS Maker project is one resource that belongs to an organization:
a table with `organization_id`, a paginated CRUD API, and a page in the tenant app.
You do not invent the structure: you **copy the `projects` reference module** and rename.
It lives on the `reference/projects` branch of each repo, never on `main`, so a generated
project does not have it locally. Fetch the files from GitHub as you go.

> Source of truth — read the file you are about to copy, don't reproduce it from memory.
> Base URL: `https://raw.githubusercontent.com/SaaS-Maker-Stack/<backend|frontend>/reference/projects/`
>
> Backend: `app/models/project.py`, `app/schemas/project.py`, `app/services/project_service.py`,
> `app/controllers/projects.py`, `alembic/versions/c3d4e5f6a7b8_add_projects_table.py`,
> `tests/test_projects.py`.
>
> Frontend: `src/types/api.ts` (bottom), `src/hooks/useProjects.ts`, `src/hooks/useDebounce.ts`,
> `src/components/features/projects/{status.ts,ProjectStatusBadge.tsx,ProjectList.tsx,ProjectFormDialog.tsx,DeleteProjectDialog.tsx}`,
> `src/pages/ProjectsPage.tsx`, `src/pages/ProjectsPage.test.tsx`.
>
> Diffs of the wiring: `git diff main..reference/projects` in each repo shows exactly which
> existing files change (router, sidebar, types, models `__init__`, `alembic/env.py`, `main.py`).

## Fast path: the CLI

If the project has the `generator:*` anchors (any project scaffolded with `saas-maker new`,
or the template since Sep 2026), generate the whole module and then only fill in the business
logic:

```bash
uvx saas-maker generate module invoice --label Factura --label-plural Facturas --feminine \
  --fields "number:str:Número,amount:money:Monto,notes:text?:Notas,due:date?:Vence" \
  --status "draft=Borrador,sent=Enviada,paid=Pagada"
```

Kinds: `str text int float money bool date datetime choice(a=Label|b=Label)` (`?` = optional;
the first field is the title). Use `money` for amounts (never `float`) and `choice` for closed
lists that are not the row's status.
It writes both sides, the migration on the current alembic head and the tests, and formats the
backend with ruff. Then run the ship checklist below. Use the manual checklist when the shape
differs from a plain CRUD (nested resources, extra endpoints, custom permissions).

## Before writing anything

1. Read the project's `DESIGN.md` (repo root) and both `CLAUDE.md` files (backend, frontend).
2. Decide the names once and stick to them (example for `invoice`):

| Concept | Backend | Frontend |
|---|---|---|
| Model / type | `Invoice` in `app/models/invoice.py` | `InvoiceResponse`, `InvoiceCreate`, `InvoiceUpdate`, `InvoiceListResponse`, `InvoiceListParams` |
| Table / prefix | `invoices`, router prefix `/invoices` | route `/invoices` |
| Service / hook | `app/services/invoice_service.py` | `src/hooks/useInvoices.ts`, query key `['invoices', params]` |
| Controller / page | `app/controllers/invoices.py` | `src/pages/InvoicesPage.tsx` |
| Components | — | `src/components/features/invoices/` |
| Tests | `tests/test_invoices.py` | `src/pages/InvoicesPage.test.tsx` |

3. Decide the fields, the status enum (if any) and **who writes**: default is read = any member,
   write = `require_role("admin")` (owner > admin > member). Keep that unless the feature says otherwise.

## The rule that bites: isolation by organization

- The table has `organization_id` (FK `organizations.id`, `ondelete="CASCADE"`, indexed) and
  `created_by` (FK `users.id`, `ondelete="SET NULL"`).
- **Every service function takes `organization_id` and filters by it** — list, get, update,
  delete. Never fetch by `id` alone. The controller passes `user.organization_id` from the JWT.
- A resource from another organization **answers 404, never 403**: the API must not reveal
  it exists. Services return `None`/`False`; the controller raises `NOT_FOUND`.
- The response schema does not expose `organization_id` (the session already implies it).

## Backend checklist

1. **Model** `app/models/<name>.py` — copy `project.py`. UUID pk, the two FKs above,
   `created_at`/`updated_at` via `utcnow`, `__tablename__` plural.
2. **Schemas** `app/schemas/<name>.py` — `Create` (validated lengths), `Update` (all optional),
   `Response`, `ListResponse(items, total, page, page_size)`. Status as a `Literal`.
3. **Service** `app/services/<name>_service.py` — `list_*(session, organization_id, *, q, status,
   page, page_size) -> (items, total)` (ilike on the searchable field, order `created_at desc, id desc`),
   `get_*`, `create_*(session, organization_id, created_by, **fields)`, `update_*` (bumps `updated_at`),
   `delete_*`.
4. **Controller** `app/controllers/<plural>.py` — `GET ""` (query params `q`, `status`, `page>=1`,
   `page_size 1..100`), `GET /{id}`, `POST ""` (201), `PUT /{id}`, `DELETE /{id}` (204).
   Write routes take `dependencies=[Depends(require_role("admin"))]`. `detail` messages in Spanish.
5. **Register** at the anchors:
   - `app/models/__init__.py`: import + entry in `__all__` (`# generator:models`).
   - `alembic/env.py`: add the model to the import block so autogenerate sees it.
   - `app/main.py`: `app.include_router(<plural>.router, prefix="/<plural>", tags=["<plural>"])`
     right before `# generator:tenant-routers` (if the anchor is missing, after the last tenant router).
6. **Migration**: `make makemigrations m="add <plural> table"`, then **open the file** and check the
   FKs keep `ondelete`, the `organization_id` index exists, and `down_revision` is the current head.
   `make migrate`.
7. **Tests** `tests/test_<plural>.py` — copy `test_projects.py`. Keep the helpers (`_add_member`
   invites + accepts; `_create`). Minimum coverage: create+get, paginated/search list, update, delete,
   validation 422, member gets 403 on write, **another org gets 404 on get/put/delete and an empty list**,
   `switch-organization` changes what the same user sees.
8. `make test` and `make lint`. Update the endpoints list in `backend/CLAUDE.md`.

## Frontend checklist

1. **Types** at the bottom of `src/types/api.ts` before `// generator:types`: `<Name>Status`,
   `<Name>Response`, `<Name>ListResponse`, `<Name>ListParams`, `<Name>Create`, `<Name>Update`.
2. **Hook** `src/hooks/use<Plural>.ts` — copy `useProjects.ts`: one query (`['<plural>', params]`,
   `placeholderData: (previous) => previous`) and three mutations that invalidate `['<plural>']`.
   Reuse `useDebounce` (create it from the reference if the project lacks it).
3. **Feature components** `src/components/features/<plural>/`:
   - `status.ts` — Spanish labels + the ordered status list (constants live here, not in a
     component file: the `react-refresh/only-export-components` lint rule).
   - `<Name>StatusBadge.tsx` — semantic tokens only (`bg-success/10 text-success`, `info`, `muted`).
   - `<Name>List.tsx` — the **four states** from DESIGN.md: skeleton rows while loading, empty state
     with CTA (only when `canWrite && !hasFilters`; "Sin resultados" when filtering), error card with
     "Reintentar", populated table with pagination ("Página X de Y", Anterior/Siguiente). Row actions
     (Editar / Eliminar) in a dropdown with `aria-label="Acciones de <name>"`, only when `canWrite`.
   - `<Name>FormDialog.tsx` — react-hook-form + zod, one dialog for create and edit (`item: T | null`),
     reset on open, trims and nulls empty optional text.
   - `Delete<Name>Dialog.tsx` — AlertDialog, destructive action.
4. **Page** `src/pages/<Plural>Page.tsx` — heading, subtitle "Los <plural> de {organization_name}",
   primary CTA "Nuevo <name>" for owner/admin (`canWrite`), debounced search with `aria-label`,
   status `Select` with an "all" option, the list, both dialogs. Members see the list without actions.
5. **Wire** at the anchors: `src/router/index.tsx` (`// generator:route-imports`, `// generator:routes`
   inside the protected `AppLayout` group) and `src/components/layout/Sidebar.tsx` (`// generator:nav`,
   pick a `lucide-react` icon; add `minRole` only if reading is restricted).
6. **Test** `src/pages/<Plural>Page.test.tsx` — copy `ProjectsPage.test.tsx`: mock `@/hooks/useAuth`
   and the hook, render with `renderWithProviders` (`src/test/render.tsx`). Cover: list renders with
   status labels, empty state CTAs, member hides write actions, create dialog submits the right
   payload, delete confirmation calls the mutation.
7. `npm run lint`, `npm run typecheck`, `npm test`, `npm run build`. Update `frontend/CLAUDE.md`
   (features, hooks, query key, route).

**Copy**: Spanish, sentence case, no exclamation marks, follow DESIGN.md tone. Never hard-code a color;
if DESIGN.md names a token the project lacks, add it to `src/index.css` (light + `.dark`) first.

## Ship checklist

- [ ] Cross-org test exists and passes (404 on get/put/delete, empty list).
- [ ] Migration reviewed by hand and applied; `alembic` head is linear.
- [ ] Backend `make test` + `make lint`; frontend lint + typecheck + test + build.
- [ ] Both `CLAUDE.md` updated; anchors still in place for the next module.
- [ ] Walked the page in the browser: loading, empty, error, populated, light and dark.

## Anti-patterns

- ❌ `select(Model).where(Model.id == id)` without `organization_id` — the isolation leak.
- ❌ Returning 403 for a resource of another organization — it must be 404.
- ❌ Filtering by organization in the controller instead of the service — services are the boundary.
- ❌ `create_all` or hand-editing the DB — Alembic only; review the autogenerated file.
- ❌ Merging the `reference/projects` branch into the project — it is a template to copy, not a dependency.
- ❌ Touching the admin panel for a tenant module — platform admins do not manage tenant data.
- ❌ Hard-coded colors, English UI copy, or a page without the four states.
