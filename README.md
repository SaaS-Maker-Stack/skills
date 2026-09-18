# SaaS Maker Skills

[![skills.sh](https://skills.sh/b/SaaS-Maker-Stack/skills)](https://skills.sh/SaaS-Maker-Stack/skills)

[Agent Skills](https://agentskills.io) that teach **any** skills-compatible coding
agent (Claude Code, Cursor, Codex, Gemini CLI, …) how to work with
[SaaS Maker](https://github.com/SaaS-Maker-Stack/saas-maker) — the multi-tenant SaaS
starter (FastAPI + SQLModel + PostgreSQL, React 19 + shadcn/ui, admin panel, Kamal).

The skills are **thin pointers**: they carry the procedure and the non-negotiable rules,
and tell your agent which file of the project or of the template to read on demand.

## Skills

| Skill | Use it when |
|-------|-------------|
| **saas-maker-primer** | First — the three services, the tenant-isolation rule, quality gates, how to apply an external `DESIGN.md`, and which skill to load next. |
| **saas-maker-cli** | Creating a project (`uvx saas-maker new`) or generating a CRUD module (`saas-maker generate module`). |
| **saas-maker-add-module** | Adding a tenant-scoped resource by hand or adapting a generated one: backend and frontend checklists, the `projects` reference module. |
| **saas-maker-deploy** | Deploying or operating on a server with Kamal 2: order, migrations, first admin, day-2 commands. |

## Install

### `npx skills` (any Agent-Skills client)

```bash
# this project
npx skills add SaaS-Maker-Stack/skills --skill '*' --yes

# all projects
npx skills add SaaS-Maker-Stack/skills --skill '*' --yes --global

# pin to a specific agent (e.g. Claude Code)
npx skills add SaaS-Maker-Stack/skills --agent claude-code --skill '*' --yes
```

### Claude Code plugin

```bash
/plugin marketplace add SaaS-Maker-Stack/skills
/plugin install saas-maker-skills@saas-maker-skills
```

## Versioning

Skills are versioned in lockstep with the template: the file pointers inside each skill
pin the template release they target (currently **v0.1.0**). Install the skills release
that matches the template version you are building on.

## License

Apache-2.0
