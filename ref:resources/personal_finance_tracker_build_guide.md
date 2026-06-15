# Building "Finance Buddy" — A Real Personal Finance Tracker

> From a static dashboard to a live, self-hosted product on `vishalkatariya.dev`
> Part of a larger personal section: **Expenses · Notes · Bookmarks**

---

## 1. The vision

Right now you have a beautiful static HTML dashboard fed by a one-time CSV export from Buddy. The goal is to turn this into a **live personal product** — three connected modules under one roof:

```
vishalkatariya.dev/me
├── /finance      → expense + income tracker (this guide's focus)
├── /notes        → Notion-style notes (your existing Notion workspace, mirrored)
└── /bookmarks    → link saver with tags + previews
```

All three share one auth layer, one design system, one backend, one database. This guide focuses on **finance** but designs the foundation so notes and bookmarks slot in later.

---

## 2. Recommended stack

This aligns with your existing portfolio stack (SvelteKit + Fastify + TypeScript) and your homelab.

| Layer | Choice | Why |
|---|---|---|
| Frontend | **SvelteKit** + TypeScript | Already your portfolio stack; SSR + API routes in one |
| Styling | **Tailwind** + your design tokens | Reuse the NothingOS/teal system you already built |
| Charts | **Chart.js** or **LayerChart** | Chart.js is what the dashboard already uses |
| Backend API | **Fastify** + TypeScript | Already your stack; fast, typed, plugin ecosystem |
| Database | **PostgreSQL 16** | Relational fits transactions perfectly; runs on `atlas` |
| ORM | **Drizzle ORM** | Type-safe, lightweight, SQL-first (better than Prisma for this) |
| Auth | **Lucia** or single-user JWT | It's just you — keep it simple |
| Validation | **Zod** | Shared schemas between front and back |
| Deploy | **Docker Compose** on `athena` | Bind mounts under `/srv/dev-shared/homelab/finance/` |
| Ingress | **Caddy** + **Tailscale** | Your existing private-access pattern |
| Public face | **Vercel** (SvelteKit) → API via Tailscale Funnel or VPS proxy | Portfolio already on Vercel |

> **Single-user shortcut:** since this is just you, you can skip a full auth provider. A long-lived signed JWT in an httpOnly cookie, gated by one password (hashed with argon2), is plenty. Add Lucia later if you ever open it up.

---

## 3. Data model

The schema below covers finance and is extensible to notes/bookmarks. Drizzle syntax:

```typescript
// schema.ts
import { pgTable, serial, text, timestamp, numeric, boolean, integer } from 'drizzle-orm/pg-core';

export const accounts = pgTable('accounts', {
  id: serial('id').primaryKey(),
  name: text('name').notNull(),          // "SBI", "N26", "Cash"
  currency: text('currency').default('EUR'),
  createdAt: timestamp('created_at').defaultNow(),
});

export const categories = pgTable('categories', {
  id: serial('id').primaryKey(),
  name: text('name').notNull(),          // "Rent", "Groceries", "Subway"
  kind: text('kind').notNull(),          // 'expense' | 'income'
  parent: integer('parent'),             // for Shopping → Electronics nesting
  color: text('color'),                  // hex for charts
  essential: boolean('essential').default(false), // for waste audit
});

export const transactions = pgTable('transactions', {
  id: serial('id').primaryKey(),
  date: timestamp('date').notNull(),
  amount: numeric('amount', { precision: 10, scale: 2 }).notNull(), // negative = expense
  note: text('note'),
  categoryId: integer('category_id').references(() => categories.id),
  accountId: integer('account_id').references(() => accounts.id),
  source: text('source'),                // "Buddy import", "manual", "bank-sync"
  avoidable: boolean('avoidable').default(false), // waste-audit flag
  createdAt: timestamp('created_at').defaultNow(),
});

export const budgets = pgTable('budgets', {
  id: serial('id').primaryKey(),
  categoryId: integer('category_id').references(() => categories.id),
  monthlyLimit: numeric('monthly_limit', { precision: 10, scale: 2 }),
});
```

The whole dashboard you have today is just **aggregations over the `transactions` table** — monthly sums, category groupings, year rollups. Every chart becomes a SQL query.

---

## 4. Backend API (Fastify)

```typescript
// routes/transactions.ts
import { FastifyInstance } from 'fastify';
import { z } from 'zod';
import { db } from '../db';
import { transactions } from '../schema';
import { sql, eq, gte, lte, and } from 'drizzle-orm';

const txSchema = z.object({
  date: z.string(),
  amount: z.number(),
  note: z.string().optional(),
  categoryId: z.number(),
  accountId: z.number(),
});

export default async function (app: FastifyInstance) {
  // List with filters
  app.get('/api/transactions', async (req) => {
    const { from, to } = req.query as { from?: string; to?: string };
    return db.select().from(transactions).where(
      and(
        from ? gte(transactions.date, new Date(from)) : undefined,
        to ? lte(transactions.date, new Date(to)) : undefined,
      )
    );
  });

  // Monthly aggregation — powers the main chart
  app.get('/api/stats/monthly', async () => {
    return db.execute(sql`
      SELECT to_char(date, 'YYYY-MM') AS month,
             SUM(CASE WHEN amount < 0 THEN -amount ELSE 0 END) AS expenses,
             SUM(CASE WHEN amount > 0 THEN amount ELSE 0 END) AS income
      FROM transactions
      GROUP BY month ORDER BY month;
    `);
  });

  // Create
  app.post('/api/transactions', async (req, reply) => {
    const data = txSchema.parse(req.body);
    const [row] = await db.insert(transactions).values({
      ...data, date: new Date(data.date),
      amount: String(data.amount),
    }).returning();
    return reply.code(201).send(row);
  });
}
```

Every "tab" in your current dashboard maps to one stats endpoint:

| Tab | Endpoint |
|---|---|
| Overview | `/api/stats/monthly` + `/api/stats/categories` |
| Budget | `/api/stats/avg-monthly` |
| Income | `/api/stats/income-by-category` |
| Yearly | `/api/stats/yearly` |
| Waste | `/api/stats/avoidable` |

---

## 5. CSV import (the bridge from Buddy)

Keep using Buddy on your phone, export monthly, and run an import script. This is your **migration path** — no need to switch apps day one.

```typescript
// scripts/import-buddy.ts
import { parse } from 'csv-parse/sync';
import { readFileSync } from 'fs';

const rows = parse(readFileSync('buddy_export.csv'), {
  delimiter: ';', columns: true,
});

for (const r of rows) {
  const amount = parseFloat(r.Amount.replace(',', '.'));
  // upsert category by name+kind, then insert transaction
  // dedupe on (date, amount, note) so re-imports are idempotent
}
```

**Idempotency tip:** dedupe on a hash of `(date, amount, note)` so you can re-import the same month safely without doubling entries.

---

## 6. Frontend (SvelteKit)

Your existing HTML/Chart.js translates almost directly. SvelteKit load functions fetch from the API:

```typescript
// routes/me/finance/+page.ts
export async function load({ fetch }) {
  const [monthly, categories] = await Promise.all([
    fetch('/api/stats/monthly').then(r => r.json()),
    fetch('/api/stats/categories').then(r => r.json()),
  ]);
  return { monthly, categories };
}
```

Drop your existing chart code into Svelte components. The `bld()` functions you wrote become `onMount` handlers per component.

---

## 7. Deployment on your homelab

Follow your established pattern (bind mounts, Tailscale, Caddy):

```yaml
# /srv/dev-shared/homelab/finance/docker-compose.yml
services:
  finance-db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: finance
      POSTGRES_PASSWORD_FILE: /run/secrets/db_pw
    volumes:
      - ./data:/var/lib/postgresql/data   # bind mount, your rule
    restart: unless-stopped

  finance-api:
    build: ./api
    ports:
      - "127.0.0.1:7100:3000"             # bind localhost only, your rule
    depends_on: [finance-db]
    restart: unless-stopped
```

Then a Caddy block on `athena` (file-cert pattern from your Rock 5T blueprint):

```
finance.auxois-wyrm.ts.net {
    bind <tailscale-ip>
    tls /etc/caddy/certs/...crt /etc/caddy/certs/...key
    reverse_proxy 127.0.0.1:7100
}
```

For the **public** portfolio version on Vercel, the SvelteKit frontend can call the homelab API over **Tailscale Funnel** (exposes one service publicly) or you keep finance behind Tailscale and only the marketing/portfolio pages public. Given the sensitivity of financial data, **keep `/me/finance` private (Tailscale-only)** and only expose the public portfolio shell.

---

## 8. Phased roadmap

**Phase 1 — Foundation (weekend 1)**
- Postgres + Drizzle schema on `athena`
- Buddy CSV importer (idempotent)
- Fastify with 3 read endpoints (monthly, categories, yearly)

**Phase 2 — Port the dashboard (weekend 2)**
- SvelteKit app at `/me/finance`
- Move existing Chart.js tabs to live data
- Single-password auth (argon2 + JWT cookie)

**Phase 3 — Make it interactive (weekend 3)**
- Add/edit/delete transactions in-app
- Budget limits + "over budget" alerts
- Waste-audit flagging UI (toggle `avoidable` per transaction)

**Phase 4 — The personal section (later)**
- `/me/notes` — pull from Notion API (you already have the MCP/workspace)
- `/me/bookmarks` — link saver with OpenGraph preview fetching
- Shared nav shell, shared auth, one design system

**Phase 5 — Automation (stretch)**
- Bank sync via GoCardless / Nordigen API (free tier, EU PSD2) — auto-import transactions, no more manual CSV
- Monthly email summary (your homelab can run a cron + a mailer)

---

## 9. Why this fits your goals

- **Portfolio piece:** a full-stack, self-hosted product with real data is far stronger than a static site — shows backend, DB design, deployment, and design sense in one.
- **Reuses your stack:** SvelteKit + Fastify + Postgres + Docker + Caddy + Tailscale are all things you already run. Nothing new to learn from scratch.
- **MLOps angle:** once you have a clean transactions table, a "spending forecast" or "anomaly detection on unusual spend" becomes a natural ML mini-project for your portfolio.
- **Genuinely useful:** unlike a toy project, you'll actually use this every month — which means you'll maintain it, which means it stays a living showcase.

---

## 10. First concrete step

```bash
mkdir -p /srv/dev-shared/homelab/finance/{api,data}
cd /srv/dev-shared/homelab/finance
# scaffold
npm create svelte@latest web
cd api && npm init -y && npm i fastify drizzle-orm postgres zod
npm i -D drizzle-kit tsx typescript
```

Then port the schema from §3, write the importer from §5, point it at your `buddy_export.csv`, and you have a live database by end of day one.

---

*Generated as a companion to the Finance Buddy dashboard · vishalkatariya.dev*
