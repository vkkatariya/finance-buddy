# CLAUDE.md
> This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.
> Project-specific instructions. Workflow conventions live in `./workflow/`.

---

## Session startup

Before any meaningful work, read these files in order:

1. `CONTEXT.md` — stack, data sources, database schema, design system, hard constraints
2. `tasks/DEVLOG.md` (last 3 entries) — current world state
3. `tasks/todo.md` — phased task list
4. `tasks/lessons.md` — active prevention rules

**This project runs in the dual-session model** (see `workflow/SESSION-WORKFLOW.md` v2):

- **Local session** (this is normally the one you are): `[finance-buddy]-local`
  - Host: athena tmux
  - Process: Claude CLI on athena, started with `claude --remote-control '[finance-buddy]-local'`
  - Branch: `claude/local` (work on `feat/<task>` sub-branches off it)
  - Filesystem: full local access (athena's `/home/radxa/...`, Docker, Postgres on atlas via Tailscale)
  - Use for: design iteration, file edits, debugging, SvelteKit dev server, docker compose, anything needing local FS or DB
- **Cloud session** (sibling, long-lived): `[finance-buddy]-cloud`
  - Host: Anthropic cloud container (NOT your machine)
  - Process: started from https://claude.ai/code → Code tab → New session → pick repo → pick branch `claude/cloud`
  - Branch: `claude/cloud` (work on `feat/<task>-cloud` sub-branches off it)
  - Filesystem: **ONLY the GitHub repo** (no `~/dev-shared/`, no Docker, no local Postgres, no local servers)
  - Use for: `pnpm install`, `pnpm run build`, `pnpm test`, full e2e audit, anything needing CPU isolation
  - **Cannot do:** access local Postgres (no DB on cloud), access local server (no Tailscale), see raw finance CSV (gitignored), run Docker

**One-time setup** (run on first session per project, on athena):
```bash
cd ~/dev-shared/projects/finance-buddy
git branch claude/local main
git branch claude/cloud main
git push origin claude/local claude/cloud
```

**How to start the cloud session:** open https://claude.ai/code → Code tab → New session → pick `vkkatariya/finance-buddy` → pick branch `claude/cloud` → rename to `[finance-buddy]-cloud`. NOT from the CLI — `claude --remote-control` is a relay, not a cloud container.

**Cross-session handoff:** read top 3 of `tasks/DEVLOG.md` on every resume — the cloud and local sessions log to the same DEVLOG with `cloud-session-start` / `cloud-session-end` / `local-session-handoff` markers so the other side knows what happened. Coordination also happens via git branches and PRs.

## Workflow references (symlinked, homelab-only)

The `./workflow/` directory is a symlink to `~/dev-shared/workflow/` — same path on every machine via mutagen sync. **Do not commit it** (already in `.gitignore`). Read workflow files on demand, not at every session start:

- `./workflow/SESSION-WORKFLOW.md` — Claude Code session lifecycle, dual-session (local relay + Anthropic cloud), lineage branches, /remote-control, compaction
- `./workflow/AI-ROUTING.md` — L1/L2/L3 layer model, tool vs agent routing
- `./workflow/GIT-GITHUB-BLUEPRINT.md` — branch/commit/PR conventions
- `./workflow/agents_workflow/AI-AGENTS-ORCHESTRATION.md` — sub-agent dispatch patterns

If the symlink is broken on a fresh clone, recreate it:
```bash
ln -sf ~/dev-shared/workflow ./workflow
```

## Compaction + /remote-control (project-specific)

- **/remote-control** is the real command (not `/rc` — that's hallucinated)
- **Before manual /compact:** commit, push, append your work-in-progress to `tasks/DEVLOG.md`
- **After /compact** or /clear: re-read top 3 of `tasks/DEVLOG.md`, re-read this CLAUDE.md, check `git status` to reconstruct in-flight work
- **Pro plan 5hr rolling limit:** dual session uses 2 surfaces in parallel. Heavy CPU work (Playwright, builds) in cloud; interactive work in local

---

## Commands

### Local dev (Phase 2 — SvelteKit, not started)
```bash
pnpm install              # install SvelteKit deps
pnpm run dev              # SvelteKit dev server
pnpm run build            # production build
pnpm run preview          # preview production build
```

### Local dev (Phase 1 — HTML prototype, current)
```bash
open prototypes/finance-buddy-dashboard.html
# Or serve it:
python3 -m http.server 8765
# Then open http://localhost:8765/prototypes/finance-buddy-dashboard.html
```

### Backend (when Phase 2 starts)
```bash
cd /srv/dev-shared/homelab/finance/api
pnpm install
pnpm run dev
docker compose up -d postgres   # if not already running
```

### CSV import (data pipeline)
```bash
python scripts/import-buddy.ts \
  --input data/raw/transactions.csv \
  --output data/processed/transactions.jsonl
```

### CI
- Lint + typecheck + build on every push to `main`
- Workflow: `.github/workflows/ci.yml` (lint + typecheck + build)
- Frontend/backend jobs skip if directories don't exist yet (Phase 2 not started)

---

## Architecture

### Current phase: Phase 1 (HTML prototype complete), Phase 2 (SvelteKit + Postgres) not started

**Single source of truth:** `prototypes/finance-buddy-dashboard.html` — fully functional self-contained dashboard with 7 tabs.

### Production stack (target)
| Layer | Tech | Notes |
|---|---|---|
| Frontend | SvelteKit + TypeScript | SSR + API routes |
| Styling | Tailwind + custom tokens | Reuse NothingOS design tokens |
| Charts | Chart.js 4.x | Direct port from prototype |
| Backend | Fastify + TypeScript | REST API |
| ORM | Drizzle ORM | Type-safe, SQL-first |
| DB | PostgreSQL 16 | On `atlas` (Proxmox, LVM-Thin) |
| Auth | Single-user JWT | httpOnly cookie, argon2id, no OAuth |
| Deploy | Docker Compose on athena | Bind mounts only (no named volumes) |
| Ingress | Caddy + Tailscale | File-cert pattern, Tailscale-only |

### Data
- **Source:** Buddy expense tracking app CSV exports (semicolon-delimited)
- **Volume:** 993 transactions, Oct 2022 – May 2026 (44 months)
- **Privacy:** NEVER public. Tailscale-only via `finance.auxois-wym.ts.net`
- **Raw CSVs gitignored** — never commit transaction data

### Design system (port from prototype)
```css
--bg: #1B1C1D;  --surf: #252527;  --acc: #00D4AA;  --red: #F05C5C;
--glass: rgba(27,28,29,.75);  --glass-b: rgba(241,240,246,.12);
```
**Typography:** `DM Mono` (data) + `Syne 800` (display KPIs)
**Glass formula:** `backdrop-filter: blur(32px) saturate(180%)` + specular inset
**Icons:** 34-symbol inline SVG set

### Branch strategy
- `main` — trunk
- `feat/<slug>` — feature branches off `main`
- Conventional commits: `feat:`, `fix:`, `docs:`, `chore:`

### Caddy config pattern
```caddyfile
finance.auxois-wyrm.ts.net {
    bind <tailscale-ip>
    tls /etc/caddy/certs/finance.crt /etc/caddy/certs/finance.key
    reverse_proxy 127.0.0.1:7100
}
```

---

## Hard constraints

- **Finance data is PRIVATE.** Tailscale-only. Never public. Never commit raw CSV.
- **Bind mounts only** — no named Docker volumes
- **Always `cd` into service dir** before `docker compose` commands
- **Tailscale cert key ownership:** `root:caddy 640`
- **Keep finance separate** from the public portfolio — different Tailscale subdomain
- **No over-engineering** — one user, keep auth simple (single JWT, no multi-tenant)

---

## Task management

Agents write to `tasks/DEVLOG.md` (newest at top) at the end of every session. Format and required fields are in `AGENTS.md`. This is mandatory — a missing DEVLOG entry breaks the next session's handoff.

Lessons from corrections go in `tasks/lessons.md` (newest at top, numbered L-0NN).

The `ref:resources/` directory exists in this project — note the colon in the name can confuse some git commands; use `--find-renames` when moving files in/out.
