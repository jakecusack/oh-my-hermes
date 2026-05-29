# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

Oh My Hermes is an opinionated extension layer for [Hermes Agent](https://hermes-agent.nousresearch.com) — like Oh My Zsh is to Zsh. **It is not a runtime, server, or application.** It is a curated collection of markdown files (skills, agent role definitions, workflows), starter templates, and shell scripts. The repository ships files; Hermes runs them.

There is **no build step, no test framework, no package manifest, no application code** here (the only `.ts`/`.js` files are health-endpoint templates that get copied into *other* projects). Treat this as a documentation-and-file-distribution project.

## Commands

```bash
bash scripts/verify.sh      # Checks files are installed correctly into ~/.hermes/ (run after install.sh)
bash docker/test.sh         # Smoke tests — validates structure WITHOUT a live Hermes session; this is the closest thing to a unit test suite
```

`docker/test.sh` is the validation gate to run after editing skills, agents, or workflows. It verifies every expected file exists in `~/.hermes/` **and enforces the skill-description convention** (see below). Build/run a full container with `docker-compose up` or via `Dockerfile` (only needed to test the install flow end-to-end).

The install/distribution scripts (do not run these against this repo's own directory — `bootstrap.sh` guards against it):

```bash
bash install.sh             # Copies skills/, workflows/, agents/ → ~/.hermes/
bash scripts/bootstrap.sh   # Run from a TARGET project: scaffolds AGENTS.md, .env.example, health route
bash scripts/setup-cto.sh   # Creates Hermes profiles, inits kanban, schedules crons (needs GITHUB_TOKEN)
bash scripts/uninstall.sh   # Removes Oh My Hermes files from ~/.hermes/
```

## Architecture

Hermes is a long-lived operator (runs 24/7 on a VPS) that talks to a founder over chat (Telegram/Slack/etc.), remembers context across sessions, and orchestrates work by **loading skills on demand**. Oh My Hermes supplies those skills plus a multi-agent "CTO loop".

Four kinds of content, each installed to a matching directory under `~/.hermes/`:

- **`skills/`** → `~/.hermes/skills/` — Individual capabilities in [agentskills.io](https://agentskills.io) SKILL.md format. Hermes matches a skill to a task using its `description` frontmatter field, then follows its Procedure. This is the primary unit of work in the repo.
- **`agents/`** → `~/.hermes/agents/` — Role definitions (cto, pm, dev, qa, ops, security) for the autonomous CTO loop. Each agent owns kanban columns and composes skills.
- **`workflows/`** → `~/.hermes/workflows/` — Composite documents that chain multiple skills into a lifecycle (e.g. `idea-to-deploy`, `cto-loop`).
- **`templates/`**, **`examples/`** — Starter files (`AGENTS.md.template`, `.env.example`, health endpoints) that `bootstrap.sh` copies into *new* projects, plus a working `starter-app`.

The CTO loop (agents + kanban + crons) is the headline feature: GitHub issue → PM triages → Dev implements → Security scans → QA reviews → founder approves via chat → Ops deploys & monitors. Flow lives in `workflows/cto-loop.md` and `docs/architecture.md`.

Default stack is opinionated but pluggable: **Vercel** (deploy), **Supabase** (Postgres/auth, RLS for isolation), **Sentry + Uptime Kuma** (monitoring), **Slack webhook** (notifications). Every project is expected to expose a `/api/health` endpoint returning `{ status, version, timestamp }` (200 = healthy, 503 = unhealthy).

`AGENTS.md` is the canonical contributor guide for working *inside this repo*; the deeper rationale (engine routing, defaults) is in `docs/`.

## Conventions

**Skill file format** — every file in `skills/` must follow the agentskills.io standard with frontmatter (`name`, `description`, `version`, `tags`) and these sections: `## Overview` / `## When to Use` / `## Prerequisites` / `## Procedure` / `## Pitfalls` / `## Verification`.

**Skill `description` is load-bearing and machine-checked** — it MUST start with `Use when...` and describe *only triggering conditions*, never a workflow summary. `docker/test.sh` fails the build if any skill description doesn't start with "Use when". This field is how Hermes decides whether to load the skill.

**Content quality rules** (from `AGENTS.md`):
- Procedures must be specific and testable, not aspirational.
- Pitfalls must be *observed* failure modes only — no hypotheticals.
- Prerequisites must list every credential and tool required.
- Verification must be a concrete check, not "confirm it looks right".
- Keep skills concise (target under ~350 words) for Hermes memory efficiency.
- No marketing language in technical docs; unbuilt features must be labeled V2/V3 in the roadmap.

**Scripts** must be idempotent (safe to re-run), check prerequisites first, and print clear `[OK]`/`[MISSING]`/`[ERROR]` status lines.

**Commits** — one commit per meaningful change. Never bundle a skill change with a doc change.

**Engine routing decisions** (`docs/engines.md`) and the Vercel+Supabase defaults (`docs/architecture.md`) are research-based and opinionated by design. Do not change routing recommendations without reading that document first.

**Stay in sync** — `scripts/verify.sh`, `docker/test.sh`, and the README badges/tables hardcode the lists of skills/agents/workflows and their counts. When you add or remove a skill/agent/workflow, update all of them together (the CHANGELOG shows counts have drifted before).
