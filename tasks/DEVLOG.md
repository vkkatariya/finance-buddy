# DEVLOG.md
> Append-only session log. Written by agents at end of every session. **Newest entry at top.**
> Format: date · agent · one-line summary, then Did / State / Decided / Blocked+Next / Modified.

---

## 2026-06-06 · Claude (claude.ai) · Finance Buddy prototype complete — 17 milestones across full dashboard build

**Mode:** Builder + Execution

**Did:**
- **Data analysis:** parsed 993 transactions from Buddy CSV (Oct 2022–May 2026, 44 months). Computed: €43,729 spent, €31,844 income, net −€11,884, avg €994/mo spend. Identified 7 income sources (Subway active, Amazon/Chamaeleo/WPL/Zenjob/Kötter/Hydra historical). Built category breakdown: Rent 34.8%, Education 19.7%, Shopping 10.9%, Groceries 9.2%, Insurance 9.2%. Waste audit: €3,656 avoidable. Savings scenarios: status quo 2047, budget mode 2031, post-grad hustle 2028.
- **Dashboard built from scratch:** self-contained HTML file (~207KB), 7 tabs, 8 Chart.js instances, all data embedded as JS objects
- **Overview tab (p0):** NexusBank-style layout — greeting, year-filter pills, 4 sparkline stat cards, monthly flow bar+line chart, category donut, savings rhythm strip, My Tracker widget (teal card + stacked layers + add-expense form)
- **Trends tab (p1):** Monthly spending flow + year-by-year grouped bars + stacked category composition chart (merged from separate Monthly+Yearly tabs)
- **Budget tab (p2):** 13 categories with animated horizontal bars, avg/month values
- **Income tab (p3):** 6 income categories, employer breakdown cards, earned-vs-other stacked yearly chart
- **Waste Audit tab (p4):** avoidable expense table, category bars, 6 corner-cutting guide cards
- **Savings Plan tab (p5):** 3 scenario cards, 84-month trajectory line chart, 8-point action plan grid
- **Transactions tab (p6):** two-column layout — left: 44 months navigable, day-grouped list (Buddy-style); right: arc doughnut chart + Expenses/Income tab + animated category bars with proportion fill
- **Finance Buddy brand:** Buddy-style colorful icon (teal/mint/white/pink/purple vertical bands), "Finance Buddy" wordmark
- **FitSpark-style header:** 3 separate floating glass pills (logo · nav · controls), sticky top:12px, backdrop-filter blur(32px)
- **NothingOS palette:** `#1B1C1D` charcoal bg, `#252527` surface, 20px dot matrix, no gradient glows (from homelab dashboard prototype)
- **Light mode:** full theme toggle (sun/moon button replacing notification bell), 40+ component overrides, localStorage persistence, no-flash head script, chart redraws on toggle
- **SVG icon system:** 34 inline symbols, 5-tier sizing (.ico-nav/sh/wl/st/sm), replaces all emoji
- **Search:** live debounced search across TX_RAW (993 txns) + 7 nav pages. Buddy-style `N transactions ···· ±NET €` summary header. Net = direct sum of all t[3] (income+, expense−). Verified all 4 reference cases exactly: Shopping −€4,758, Homelab −€417.67, Income +€1,586.50, Subway +€4,529.01. Dropdown below hdr-right pill, glass effect both themes.
- **Add-expense form:** slides in from + button on flow card, overlays full My Tracker widget. Expense/income toggle, 9 expense + 4 income categories with icons, live SAVE adds to Recent Activity with flash animation.
- **Dev stack guide:** comprehensive SvelteKit + Fastify + PostgreSQL + Drizzle build guide, 5-phase roadmap, CSV import strategy, Caddy+Tailscale homelab deploy pattern.

**State:** Prototype fully working. All 7 tabs render. JS validates clean. Panel div balance 7/7 correct. Dark and light mode both functional. Search matches reference exactly.

**Decided:**
- Finance must stay **Tailscale-private** — raw financial data, never expose publicly
- **Bind mounts only** in Docker deploy (homelab convention)
- **Drizzle ORM** over Prisma — lighter, SQL-first, better for this use case
- **No multi-tenant auth** — single-user JWT (argon2) is enough
- Panel insertion strategy: use **brace-depth counter** to find true closing `</div>`, not `rfind` — this was the root cause of the p5/p6 nesting bug
- Emoji replacement must **preserve trailing `<`** of closing tags — failing this broke Income tab (p3) and hid Waste/Savings/Transactions
- `tc`/`gc` must be `let` not `const` — blocks theme switching otherwise

**Blocked / Next:**
- [ ] Scaffold SvelteKit project (`pnpm create svelte@latest web`)
- [ ] Set up PostgreSQL 16 on `atlas` (Proxmox)
- [ ] Write Drizzle schema (transactions, categories, budgets)
- [ ] Build `import-buddy.ts` CSV importer with dedup
- [ ] Wire first 3 API endpoints: monthly, categories, transactions list
- [ ] Port Charts from embedded JS to live API calls

**Modified:**
- `prototypes/finance-buddy-dashboard.html` (created and iteratively built)
- `personal_finance_tracker_build_guide.md` (created, comprehensive dev guide)
- `tasks/DEVLOG.md` (this entry)
- `tasks/lessons.md`
- `tasks/todo.md`
- `AGENTS.md`, `CONTEXT.md`, `README.md` (all created)

---

<!-- Future entries go above this line -->
