# Finance Buddy

Personal finance tracker and dashboard — part of the `vishalkatariya.dev/me` personal section.

Built from 993 real transactions (Oct 2022 – May 2026) exported from the Buddy app.

## Current state
Fully functional self-contained HTML prototype at `prototypes/finance-buddy-dashboard.html`.  
Production SvelteKit build is **not yet started** — see `tasks/todo.md`.

## Features (prototype)
- **Overview** — sparkline stat cards, monthly flow chart, donut breakdown, NexusBank-style My Tracker widget with live add-expense form
- **Trends** — 44-month bar chart + year-by-year comparison + stacked category composition
- **Budget** — avg monthly spend per category with animated bars
- **Income** — category breakdown, employer cards (7 employers), earned-vs-other yearly chart
- **Waste Audit** — avoidable expense table, category bars, 6 corner-cutting guide cards
- **Savings Plan** — 3 scenarios + 84-month trajectory chart + action plan
- **Transactions** — 44 months navigable, day-grouped list (Buddy-style), arc doughnut chart, Expenses/Income category tabs
- **Search** — live debounced search across all 993 transactions + nav pages, Buddy-style `N transactions ···· ±NET €` summary header
- **Dark/Light mode** — full theme toggle with localStorage persistence, chart redraws

## Stack
| Layer | Current (prototype) | Planned (production) |
|---|---|---|
| Frontend | Self-contained HTML | SvelteKit + TypeScript |
| Styling | Vanilla CSS (NothingOS) | Tailwind + design tokens |
| Charts | Chart.js 4.4.1 | Chart.js (reuse) |
| Data | Embedded JS (static) | PostgreSQL 16 + Drizzle ORM |
| Backend | None | Fastify + TypeScript |
| Auth | None | Single-user JWT (argon2) |
| Deploy | Static file | Docker Compose on `athena` |
| Access | Browser | Caddy + Tailscale (private) |

## Design system
- **Palette:** NothingOS dark — `#1B1C1D` bg, `#252527` surface, `#00D4AA` teal accent
- **Typography:** DM Mono (data/labels) + Syne 800 (display/KPIs)
- **Glass:** `backdrop-filter: blur(32px)` + `rgba(27,28,29,.75)` glass formula
- **Icons:** 34-symbol inline SVG set, 5-tier sizing system

## Running locally
```bash
open prototypes/finance-buddy-dashboard.html
# No build step — fully self-contained
```

## Repo
```
github.com/vkkatariya/finance-buddy   ← (to be created)
```
