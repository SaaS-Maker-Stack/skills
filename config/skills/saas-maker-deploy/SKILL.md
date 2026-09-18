---
name: saas-maker-deploy
description: "INVOKE when deploying a SaaS Maker project to a server, or operating it day-2 (ship a new version, run migrations in production, create the first platform admin, logs, rollback, shell in). Each service (backend, frontend, admin) deploys with Kamal 2 from its own config/deploy.yml, backend first because of CORS and the API URL baked into the SPAs. Covers prerequisites, the per-service config, the order, and the day-2 aliases."
---

# Deploy a SaaS Maker project (Kamal 2)

Each service deploys **independently** with **Kamal 2** from its own
`config/deploy.yml`; secrets come from its own `.kamal/secrets` (gitignored).
Postgres runs on the same server as a Kamal **accessory** of the backend
(`postgres:17`, bound to `127.0.0.1:5434`, data in `./data`).

> Source of truth: the three `config/deploy.yml` files of the project and
> https://kamal-deploy.org/docs. Read the project's files before editing; the
> template versions are at
> `https://raw.githubusercontent.com/SaaS-Maker-Stack/saas-maker-<backend|frontend|admin>/main/config/deploy.yml`.

## Prerequisites

```bash
gem install kamal        # Kamal 2, needs Ruby (https://kamal-deploy.org)
docker login             # the registry user in deploy.yml must be able to push
```

Plus: a Linux VM you can SSH into as the `ssh.user` (Docker is installed by
`kamal setup`), and DNS `A` records for `api.`, `app.` and `admin.<domain>` → the
server IP (kamal-proxy does Let's Encrypt, so DNS must resolve **before** the first
deploy). Build arch is `amd64`; on Apple Silicon Kamal builds with buildx, which works
but is slow the first time.

## What the CLI already filled in

If the project was created with `saas-maker new` and the deploy questions were
answered, this is already done: image names `<docker-user>/<slug>-api|-frontend|-admin`,
`servers.web.hosts`, `registry.username`, `ssh.user`, `proxy.host` per service,
`CORS_ORIGINS` in the backend, `POSTGRES_DB` of the accessory, the `ARG
VITE_API_BASE_URL=https://api.<domain>` default in the two SPA Dockerfiles, and the
three `.kamal/secrets` (registry password, Postgres password, JWT secret,
`FRONTEND_URL`, SMTP). With `--defaults` or skipped questions you will find the
placeholders `your-username`, `your-server-ip`, `*.example.com` and
`your_docker_password` — grep for them and replace before deploying:

```bash
grep -rn "your-username\|your-server-ip\|example.com\|your_docker_password" */config/deploy.yml */.kamal/secrets */Dockerfile
```

SMTP: the backend needs a real SMTP host in `.kamal/secrets` (SendGrid, SES, Resend,
Postmark…) or invitations, verification and password resets never arrive.

## Order matters on first install: backend → frontend → admin

1. **Backend first.** `cd backend && kamal setup` (first time: installs Docker, boots
   kamal-proxy, starts the Postgres accessory, pushes secrets, deploys). Then run the
   migrations and create the first platform admin — **neither is automatic**:
   ```bash
   kamal db-migrate                                   # alias: app exec "alembic upgrade head"
   kamal app exec -i "python scripts/create_admin.py --email you@domain.com --name 'Your Name'"
   ```
   Check `https://api.<domain>/health` (and `/docs`, the OpenAPI UI).
2. **Frontend.** `cd frontend && kamal setup`. The API URL is baked at build time from the
   Dockerfile `ARG VITE_API_BASE_URL`; if it is wrong, fix the Dockerfile default (or add
   `builder.args.VITE_API_BASE_URL` in `deploy.yml`) and redeploy — changing env at runtime
   does nothing for a built SPA.
3. **Admin.** `cd admin && kamal setup`, same rule. Log in with the admin created in step 1.

Why the order: the SPAs call `https://api.<domain>` and the backend's `CORS_ORIGINS`
must list `https://app.<domain>,https://admin.<domain>` before the browser will talk
to it. After the first install, `kamal deploy` per service in any order.

## Day 2

All aliases are defined in each `deploy.yml`:

```bash
kamal deploy                    # ship the current commit; also how env.clear / .kamal/secrets changes reach the container
kamal logs                      # app logs -f
kamal rollback <version>        # revert to a previous image (kamal app containers lists them)
kamal app-terminal              # shell in the app container (bash on backend, sh on SPAs)
kamal db-terminal               # backend only: psql on the accessory
kamal db-migrate                # backend only: alembic upgrade head  ← after every deploy with a new migration
kamal apps / ps / stats / htop  # what's running on the host
kamal accessory logs db         # Postgres accessory
```

Migration workflow for a release: deploy the backend, then `kamal db-migrate`. Write
migrations to be safe with the previous app version still running for a few seconds
(add columns nullable/with default; drop in a later release).

Backups: the accessory keeps its data in `backend/data` **on the server**; back it up
with `pg_dump` through `kamal db-terminal` or a cron on the host. Nothing in the
template does this for you.

## Conventions to respect

- **Secrets** go through `.kamal/secrets` only (never in `deploy.yml`, never
  committed). Only non-secret env goes in `env.clear`. Never paste secret values in chat.
- **One image per service**, built from the repo's `Dockerfile`; do not add build steps
  in Kamal hooks that the Dockerfile could do.
- The Postgres accessory is for small/medium deployments; for a managed database set
  `POSTGRES_HOST/PORT` in `env.clear`, the credentials in secrets, and remove the
  accessory block.
- `DEBUG: "false"` in production (it is already).

## Anti-patterns

- ❌ Deploying the SPAs before the backend exists / before CORS lists their origins.
- ❌ Forgetting `kamal db-migrate` after a deploy that adds a migration (500s on new tables).
- ❌ Expecting a runtime env var to change `VITE_API_BASE_URL` — it is a build arg.
- ❌ Deploying with DNS not yet pointing at the server (Let's Encrypt fails, proxy 502s).
- ❌ Editing secrets/env and expecting them live without a `kamal deploy` (there is no `env push` in Kamal 2).
