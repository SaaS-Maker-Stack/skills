---
name: saas-maker-cli
description: "INVOKE when using the saas-maker CLI to create or extend a SaaS Maker project: scaffolding a new multi-tenant SaaS (uvx saas-maker new — wizard, brand color, Postgres, deploy and SMTP answers, provisioning), or generating a tenant-scoped CRUD module (saas-maker generate module — the --fields/--status DSL, labels, anchors). Covers the commands, every option, what gets written, and what to do right after."
---

# saas-maker CLI

`saas-maker` scaffolds and extends a SaaS Maker project — *zero to a running
multi-tenant SaaS in one command*, plus generators à la `rails generate`. Python;
run it with `uvx` (no install). Until it reaches PyPI, run it from GitHub:

```bash
uvx --from git+https://github.com/SaaS-Maker-Stack/saas-maker-cli saas-maker --version
# once published:  uvx saas-maker --version   →  "saas-maker X.Y.Z (template vX.Y.Z)"
```

Authoritative references (read for edge cases; do not reproduce from memory):
→ https://raw.githubusercontent.com/SaaS-Maker-Stack/saas-maker-cli/main/README.md
→ https://raw.githubusercontent.com/SaaS-Maker-Stack/saas-maker-cli/main/AGENTS.md (internals, release ceremony)

The CLI **pins a template ref** (`STACK_REF`): a given CLI version scaffolds the three
service repos at that tag, so the version output shows both.

## `saas-maker new <name>` — scaffold a project

```bash
uvx saas-maker new crm-travel            # name: lowercase, dashes → slug, db name, image names
```

Runs: **preflight** (uv, node, npm, psql client, git — informative, nothing is
installed) → **wizard** → **fetch** the template at the pinned ref → **brand**
(every `saas-template` placeholder, display name, slogan, domains, `--brand-hue`) →
write `.env` (dev), `.kamal/secrets` (prod) and a root `README.md` → **provision**
(best effort: `uv sync`, `npm install` ×2, `createdb`, `alembic upgrade head`; a failed
step prints its manual command and dependents are skipped) → **git init + initial
commit per service** (three repos; the root is not a repo).

Wizard questions, in order:

| Section | Asks | Notes |
|---------|------|-------|
| Project | display name, slogan (login tagline), brand color | Color: pick a named one (indigo, blue, teal, green, orange, rose, violet) or *custom* and type a hex (`#0066ff`) or an OKLCH hue 0–360. Hex is converted to the hue; grays are rejected. |
| Postgres | local (user/password) or remote (host/port/user/password), database name | Dev DB, `<slug_with_underscores>` by default; the backend tests use `<slug>_test` (created on first run). |
| Deploy (optional) | Docker Hub user + password, domain, server IP, SSH user | Fills `config/deploy.yml` ×3, Dockerfile API URL, `.kamal/secrets`. Skip → placeholders `your-username`, `your-server-ip`, `<slug>.com`. |
| SMTP (optional) | host, port, user, password, from | Production only; dev always uses Mailpit on `localhost:1025`. |

Options:

| Option | Effect |
|--------|--------|
| `--defaults` | No wizard: indigo, local Postgres `postgres`/no password, placeholder deploy/SMTP. CI-friendly. |
| `--skip-provision` | Write files only; no uv/npm/createdb/migrate (git init still runs). |
| `--ref <tag\|branch>` | Scaffold another template ref instead of the pinned one (dev). |
| `--source <path>` | Copy a local `saas-maker` checkout instead of downloading (dev). |
| `--output <dir>` | Parent directory (default: cwd). |

After it finishes: `cd <name>/backend && make dev`, `cd ../frontend && npm run dev`,
`cd ../admin && npm run dev` (ports 8090 / 5190 / 5191). Register a user in the app
to get the first organization; the admin panel needs `python scripts/create_admin.py`
in `backend/`. A JWT secret was generated; secrets never appear in the output — don't
print them either.

## `saas-maker generate module <name>` — tenant-scoped CRUD

Run from anywhere inside the project (it walks up to find `backend/` and
`frontend/`). Name is **snake_case singular** (`invoice`, `work_order`).

```bash
uvx saas-maker generate module invoice --label Factura --label-plural Facturas --feminine \
  --fields "number:str:Número,amount:float:Monto,notes:text?:Notas,due:date?:Vence" \
  --status "draft=Borrador,sent=Enviada,paid=Pagada" --icon Receipt
```

| Option | Effect |
|--------|--------|
| `--fields "name:kind[?][:Label],…"` | Kinds `str text int float bool date datetime`; `?` = optional. **First field must be a required `str`**: it is the title, the searchable column and the delete confirmation. Reserved names: `id organization_id created_by created_at updated_at status q page page_size`. Default: `name:str:Nombre,description:text?:Descripción`. |
| `--status "value=Label,…"` | ≥2 values. Adds a `status` column (first value is the default), a list filter and a colored badge. |
| `--label`, `--label-plural`, `--feminine` | Spanish UI copy ("Nueva factura", "Factura creada"). Without them the name is humanized. |
| `--plural <snake>` | Override the English plural (table, prefix, route, file names). |
| `--icon <LucideName>` | Sidebar icon from `lucide-react` (default `FolderKanban`). |
| `--defaults` | No prompts; flags or defaults. Without `--fields` and without `--defaults` it asks. |
| `--backend-only` / `--frontend-only` | One side only (e.g. the API exists already). |

What it writes for `invoice` (all validated first, then written atomically):

| Backend | Frontend |
|---|---|
| `app/models/invoice.py`, `app/schemas/invoice.py` | `src/types/api.ts` — types before `// generator:types` |
| `app/services/invoice_service.py` (all queries filter by `organization_id`) | `src/hooks/useInvoices.ts` (+ `useDebounce.ts` if missing) |
| `app/controllers/invoices.py` (`/invoices`, read any member, write admin+) | `src/components/features/invoices/` — list, form dialog, delete dialog, status badge |
| `alembic/versions/<rev>_add_invoices_table.py` on the current head | `src/pages/InvoicesPage.tsx` + `InvoicesPage.test.tsx` |
| `tests/test_invoices.py` (CRUD, pagination, search, cross-org 404, switch-org) | `router/index.tsx` and `Sidebar.tsx` at the anchors |
| `app/main.py` router + `models/__init__.py` export at the anchors | |

Then, always:

```bash
cd backend  && make migrate && make lint && make test
cd frontend && npm run lint && npm run typecheck && npm test
```

Review the migration and the generated business rules, then commit **in each repo**
(backend and frontend are separate git repos). For a shape that isn't a plain CRUD
(nested resources, extra endpoints, custom permissions) generate the closest CRUD and
adapt it following **`saas-maker-add-module`**.

### Anchors

The generator inserts code **before** these comment lines and refuses to run if one is
missing (nothing is written in that case):

| File | Anchor |
|------|--------|
| `backend/app/main.py` | `# generator:tenant-routers` |
| `backend/app/models/__init__.py` | `# generator:models` (inside `__all__`) |
| `frontend/src/router/index.tsx` | `// generator:route-imports`, `// generator:routes` |
| `frontend/src/components/layout/Sidebar.tsx` | `// generator:nav` (inside `navItems`) |
| `frontend/src/types/api.ts` | `// generator:types` (end of file) |

Projects generated before Sep 2026 add them by hand (compare with the template on
`main`). Never move or rename them.

## Conventions to respect

- **Don't unpin casually.** `--ref main` scaffolds unreleased template code; the pinned
  ref is the tested combination.
- **Secrets live in `.env` / `.kamal/secrets`** (gitignored). The wizard writes them; the
  CLI never prints them. Rotate anything you typed into a shared terminal.
- **The CLI does not know your business rules.** Generated code is a correct, tested
  starting point that mirrors the `projects` reference module; edit the service and
  the form, keep the isolation rule.
- Generated UI copy is Spanish; code, comments and identifiers are English.
