# CONTEXT.md — Finance Buddy
> The single source of truth for stack, data, decisions, and conventions.
> Every agent reads this before touching any file.

---

## What this is
Personal finance tracker dashboard built from real transaction data (Buddy app CSV exports).
Lives at `vishalkatariya.dev/me/finance` — Tailscale-private (never publicly accessible).
Part of a planned `/me` personal section: **finance · notes · bookmarks**.

---

## Current status
**Phase 1 — Prototype:** COMPLETE  
`prototypes/finance-buddy-dashboard.html` is a fully functional self-contained HTML dashboard.
**Phase 2 — Production SvelteKit build:** NOT STARTED

---

## Data
| Property | Value |
|---|---|
| Source | Buddy expense tracking app (Android/iOS) |
| Export format | CSV, semicolon-delimited |
| Columns | `Date;Head category;Category;Note;Amount;Currency` |
| Amount sign | Negative = expense, positive = income |
| Date range | Oct 2022 – May 2026 |
| Total transactions | 993 |
| Total months | 44 |
| Total spent | €43,729 |
| Total income | €31,844 |
| Net | −€11,884 |

**Import flow:**
```
Buddy App (phone) → CSV export → import-buddy.ts → PostgreSQL transactions table
```
Deduplication key: hash of `(date, amount, note)` — safe to re-import the same month.

---

## Stack (production target)

| Layer | Technology | Notes |
|---|---|---|
| Frontend | SvelteKit + TypeScript | SSR + API routes |
| Styling | Tailwind + custom tokens | Reuse NothingOS design tokens |
| Charts | Chart.js 4.x | Direct port from prototype |
| Backend | Fastify + TypeScript | REST API, same pattern as portfolio backend |
| ORM | Drizzle ORM | Type-safe, SQL-first, lightweight |
| DB | PostgreSQL 16 | On `atlas` (Proxmox node, LVM-Thin) |
| Auth | Single-user JWT | httpOnly cookie, argon2id hash, no OAuth needed |
| Validation | Zod | Shared schemas front ↔ back |
| Process | pm2 | Always-on for the API server |
| Deploy | Docker Compose | On `athena` (Rock 5T) |
| Ingress | Caddy + Tailscale | File-cert pattern, private only |

---

## Infrastructure

| Node | Role | Access |
|---|---|---|
| `athena` | Rock 5T, Docker host, primary SBC | LAN 192.168.178.198, Tailscale |
| `atlas` | Proxmox, PostgreSQL DB lives here | LAN 192.168.178.x, Tailscale |
| `hermes` | Mac dev workstation | Local |
| Tailnet | `auxois-wyrm.ts.net` | Private mesh |

**Docker path convention:**
```
/srv/dev-shared/homelab/finance/
├── docker-compose.yml
├── api/           ← Fastify source
└── data/          ← PostgreSQL data (bind mount)
```

**Caddy config pattern (finance private route):**
```caddyfile
finance.auxois-wyrm.ts.net {
    bind <tailscale-ip>
    tls /etc/caddy/certs/finance.crt /etc/caddy/certs/finance.key
    reverse_proxy 127.0.0.1:7100
}
```

---

## Database schema (target)

```sql
-- categories: expense/income classification
CREATE TABLE categories (
  id       SERIAL PRIMARY KEY,
  name     TEXT NOT NULL,
  kind     TEXT NOT NULL CHECK (kind IN ('expense','income')),
  parent   INTEGER,            -- for sub-categories
  color    TEXT,               -- hex for charts
  icon     TEXT                -- emoji or SVG id
);

-- transactions: core table
CREATE TABLE transactions (
  id          SERIAL PRIMARY KEY,
  date        TIMESTAMP NOT NULL,
  amount      NUMERIC(10,2) NOT NULL,  -- negative=expense, positive=income
  note        TEXT,
  category_id INTEGER REFERENCES categories(id),
  source      TEXT DEFAULT 'buddy-import',
  avoidable   BOOLEAN DEFAULT false,   -- waste audit flag
  created_at  TIMESTAMP DEFAULT NOW()
);

-- budgets: monthly targets per category
CREATE TABLE budgets (
  id           SERIAL PRIMARY KEY,
  category_id  INTEGER REFERENCES categories(id),
  monthly_limit NUMERIC(10,2)
);
```

---

## Design system

**Palette — dark (default, NothingOS):**
```css
--bg: #1B1C1D;   --surf: #252527;   --surf2: #2C2C2E;   --surf3: #303038;
--acc: #00D4AA;  --grn: #34C77B;    --red: #F05C5C;     --amb: #F5B55F;
--glass: rgba(27,28,29,.75);        --glass-b: rgba(241,240,246,.12);
```

**Palette — light (html.light class on <html>):**
```css
--bg: #F0F0EE;   --surf: #FFFFFF;   --surf2: #F4F4F2;   --surf3: #E8E8E6;
--glass: rgba(255,255,255,.72);     --glass-b: rgba(26,27,28,.1);
```

**Typography:** `DM Mono` (monospace, data/amounts) + `Syne 800` (display, KPI values)  
**Glass formula:** `backdrop-filter: blur(32px) saturate(180%)` + specular inset  
**Dot matrix:** `radial-gradient(circle, rgba(241,240,246,.04) 1px, transparent 1px)` at 20×20px  
**Icons:** 34-symbol inline SVG set, symbols in `<svg style="display:none">` block

---

## API endpoints (planned)

| Method | Path | Description |
|---|---|---|
| GET | `/api/stats/monthly` | Monthly income/expense aggregates |
| GET | `/api/stats/categories` | Category breakdown |
| GET | `/api/stats/yearly` | Year-by-year comparison |
| GET | `/api/transactions` | Paginated list with filters |
| POST | `/api/transactions` | Create transaction |
| PUT | `/api/transactions/:id` | Update transaction |
| DELETE | `/api/transactions/:id` | Delete transaction |
| POST | `/api/import/buddy-csv` | Import Buddy CSV file |

---

## Hard constraints

- **Finance data is PRIVATE.** Route must be Tailscale-only. Never public. Never commit raw CSV.
- **Bind mounts only** — no named Docker volumes (data must be inspectable without compose)
- **Always `cd` into service dir** before any `docker compose` command
- **`systemctl restart`** not `reload` when Caddy is in broken state
- **Tailscale cert key ownership:** `root:caddy 640`
- **No over-engineering** — it's one user, keep auth simple (single JWT, no multi-tenant)
- **Keep finance separate** from the public portfolio — different Tailscale subdomain

---

## File conventions

| Pattern | Usage |
|---|---|
| kebab-case | All filenames |
| `src/lib/components/` | SvelteKit components |
| `src/lib/styles/tokens.css` | Design tokens |
| `src/lib/api/` | API client helpers |
| `api/src/routes/` | Fastify routes |
| `api/src/schema.ts` | Drizzle schema |
| `scripts/import-buddy.ts` | CSV import script |

---

## Current focus
[Update this line each sprint]  
`Prototype complete. Next: scaffold SvelteKit project and wire up PostgreSQL.`
