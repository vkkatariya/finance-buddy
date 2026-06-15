# tasks/todo.md
> Current sprint items. Vishal manages this file.
> Agents: read this at session start. Mark items complete as you go.

---

## Status legend
- `[ ]` — queued
- `[~]` — in progress
- `[x]` — done
- `[!]` — blocked

---

## Phase 1 — HTML Prototype ✅ COMPLETE

- [x] Data analysis (993 txns, all categories, income sources, waste audit, scenarios)
- [x] Overview tab (sparklines, donut, monthly chart, My Tracker widget)
- [x] Trends tab (monthly + yearly merged)
- [x] Budget tab (avg monthly bars)
- [x] Income tab (categories, employers, earned-vs-other)
- [x] Waste Audit tab (table, bars, guide cards)
- [x] Savings Plan tab (3 scenarios, trajectory chart)
- [x] Transactions tab (44-month navigation, day-grouped, arc chart, Expenses/Income tabs)
- [x] Finance Buddy brand (Buddy-style icon, wordmark)
- [x] FitSpark-style floating glass header (3 pills)
- [x] NothingOS palette (homelab charcoal, no gradient glows)
- [x] Light/dark mode toggle (localStorage, chart redraws)
- [x] SVG icon system (34 symbols, 5-tier sizing)
- [x] Search (live debounced, net total header, verified vs reference)
- [x] Add-expense form (overlay, live SAVE, income/expense toggle)
- [x] Dev stack guide (SvelteKit + Fastify + Postgres build guide)
- [x] Project setup (AGENTS.md, CONTEXT.md, DEVLOG.md, lessons.md, todo.md)

---

## Phase 2 — Production Backend (next sprint)

### 2a — Database setup
- [ ] Create PostgreSQL 16 container on `atlas` (Proxmox)
  - [ ] `/srv/dev-shared/homelab/finance/` bind mount structure
  - [ ] `docker-compose.yml` (finance-db + finance-api services)
- [ ] Write Drizzle schema (`api/src/schema.ts`)
  - [ ] `categories` table (name, kind, parent, color, icon)
  - [ ] `transactions` table (date, amount, note, category_id, avoidable)
  - [ ] `budgets` table (category_id, monthly_limit)
- [ ] Run `drizzle-kit push` to create tables

### 2b — CSV importer
- [ ] Write `scripts/import-buddy.ts`
  - [ ] Parse semicolon-delimited CSV (use `csv-parse`)
  - [ ] Upsert categories by `(name, kind)` 
  - [ ] Deduplicate transactions on hash of `(date, amount, note)`
  - [ ] Test with full 993-row CSV — verify counts match prototype data
- [ ] Import historical data (Oct 2022 – May 2026)

### 2c — Fastify API
- [ ] Scaffold Fastify project (`api/`)
- [ ] `GET /api/stats/monthly` — income/expense per month
- [ ] `GET /api/stats/categories` — category breakdown (with mode=expense|income)
- [ ] `GET /api/transactions` — paginated, filterable (date range, category, search)
- [ ] `POST /api/transactions` — create (from add-expense form)
- [ ] `POST /api/import/buddy-csv` — multipart CSV upload

### 2d — Auth
- [ ] Single-user login: argon2id hash stored in env, JWT issued on correct password
- [ ] httpOnly cookie, 30-day expiry
- [ ] All routes protected except `POST /api/auth/login`

---

## Phase 3 — SvelteKit Frontend

- [ ] `pnpm create svelte@latest web`
- [ ] Install: Chart.js, Tailwind, Drizzle client, Zod
- [ ] Copy design tokens from prototype CSS (`src/lib/styles/tokens.css`)
- [ ] Port the 7 tabs as Svelte components
- [ ] `+page.ts` load functions calling API endpoints
- [ ] Add-expense form component (port from HTML prototype)
- [ ] Search component (port from HTML prototype)
- [ ] Dark/light mode: class on `<html>`, synced via localStorage

---

## Phase 4 — Deploy

- [ ] `docker-compose.yml` complete (db + api + caddy)
- [ ] `tailscale cert finance.auxois-wyrm.ts.net`
- [ ] Caddy config: `finance.auxois-wyrm.ts.net → 127.0.0.1:7100`
- [ ] pm2 or Docker restart policy for api
- [ ] Verify private access only (no public port)

---

## Phase 5 — /me integration

- [ ] Add `vishalkatariya.dev/me` route to portfolio SvelteKit
- [ ] Sub-routes: `/me/finance`, `/me/notes`, `/me/bookmarks`
- [ ] Shared auth (same JWT, shared login page)
- [ ] Shared nav shell and design system

---

## Backlog (unscheduled)

- [ ] GoCardless/Nordigen API for automatic bank sync (replaces monthly CSV export)
- [ ] Monthly email summary (cron on athena)
- [ ] Spending forecast / anomaly detection (ML mini-project)
- [ ] `/me/notes` — Notion workspace mirror
- [ ] `/me/bookmarks` — link saver with OpenGraph preview
- [ ] Budget alerts (Slack/email when over category limit)
- [ ] Penta SATA HAT for NAS — check Rock 5T compatibility before ordering
