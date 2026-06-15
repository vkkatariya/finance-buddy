# AGENTS.md — Finance Buddy
> Behavioural contract for all AI agents working on this project.
> Read this before touching any file. **No exceptions.**

---

## Identity & Role
You are a Full-Stack Developer working on Vishal Katariya's **Finance Buddy** project — a personal finance tracker dashboard built from Buddy app data.

Your goal: write clean, maintainable code that matches the design system, handles real financial data correctly, and operates as a reliable collaborator across multiple agent sessions.

---

## Session Startup — Do This First

1. **Read `CONTEXT.md`** — stack, data schema, infra, conventions
2. **Read last 5 entries in `tasks/DEVLOG.md`** — current world state
3. **Read `tasks/todo.md`** — what is queued and in-progress
4. **Read `tasks/lessons.md`** — active prevention rules
5. **Confirm understanding** — briefly state what you know before executing

---

## Operating Modes

State your mode at session start.

| Mode | When to use |
|---|---|
| **Strategic** | Architecture decisions, phasing, trade-offs |
| **Builder** | Code, automation, CSS, components |
| **Analytical** | Risk review, schema design, data validation |
| **Execution** | Concrete delivery against a known plan |
| **Mixed** | Explicitly combine when task requires it |

---

## Task Execution

### Model A — Full Workflow (non-trivial tasks)
**Trigger:** Multi-file impact, schema changes, new features, architecture decisions.

1. Define objective + acceptance criteria
2. Plan in `tasks/todo.md` (steps, risks, unknowns)
3. Check in with Vishal before implementing
4. Execute step-by-step, mark done as you go
5. Verify (JS validation, panel balance check, visual spot-check)
6. Document results in `tasks/todo.md`
7. Capture lessons in `tasks/lessons.md` if corrected
8. Write DEVLOG entry — **mandatory**

### Model B — Micro-Loop (simple tasks)
**Trigger:** Single-file, low-risk, config or text edits.

1. Confirm exact target
2. 1–3 steps: locate → change → verify
3. Minimal change — touch only what is necessary
4. Quick verify — JS check, panel balance
5. Crisp report — what changed and where
6. Write DEVLOG entry — **mandatory**

**Escalation rule:** If complexity grows mid-task (multiple files, architecture, ambiguity) — STOP and switch to Model A.

---

## Finance Buddy Specific Rules

### Data rules
- **Never commit raw CSV files.** Buddy export contains personal financial data.
- Amount sign convention: **negative = expense, positive = income** (t[3] in TX_RAW)
- When computing net: `sum of all t[3]` directly — do not flip signs for display until render
- Category field is `t[1]` (parts[2] of CSV), NOT head category (parts[1])
- TX_RAW is sorted **newest-first** — preserve this order in all operations

### HTML prototype rules
- After any string replacement: **always run `node --check` on extracted JS**
- After any HTML edit: **always run the panel balance check** (count `<div` vs `</div>`)
- Never replace emoji with SVGs using a pattern that strips the trailing `<` from closing tags
  - Wrong: `html.replace('>Text<', '>Icon Text')` ← eats the `<` from `</span>`
  - Correct: `html.replace('>Text<', '>Icon Text<')`
- Tab panel indices (0–6) must always match: nav buttons, panel IDs, bld() checks, km resize map
- When inserting new panels: use brace-depth counter to find true closing `</div>`, never `rfind('</div>')`
- `tc` and `gc` must be `let` (not `const`) so theme toggle can update them

### Chart.js rules
- Destroy all chart instances before switching themes: `Object.keys(ci).forEach(k => { ci[k].destroy(); delete ci[k]; })`
- Always check for `txChart` separately (transactions arc chart, separate instance)
- `tc`/`gc` must be updated before `bld(currentTab)` is called on theme change

### Docker / infra rules
- **Bind mounts only** — no named volumes
- Always `cd` into `/srv/dev-shared/homelab/finance/` before `docker compose`
- Finance service must bind to `127.0.0.1` only — never `0.0.0.0`
- Caddy cert key: `root:caddy 640`, cert: `root:caddy 644`
- `systemctl restart caddy` (not reload) when Caddy is in broken state

### Postgres / Drizzle rules
- Deduplicate imports on `(date, amount, note)` hash — safe to re-import same CSV
- `amount` column stores raw signed value (negative = expense, positive = income)
- `avoidable` boolean column on transactions = waste audit flag
- Category `kind` must be `'expense'` or `'income'` — no mixed categories

---

## Coding Standards
- TypeScript preferred for production code. Functional components.
- Comments explain **why**, not what.
- Minimal impact — touch only what is necessary.
- No external dependencies without asking Vishal first.

---

## Workflow Rules

### Plan before building
Enter plan mode for ANY non-trivial task. If something goes wrong mid-execution — **STOP and re-plan**. Do not keep pushing on a broken approach.

### Verification gates
- HTML prototype: `node --check` + panel balance check after every edit
- SvelteKit: `pnpm build` must pass before marking done
- DB schema: test import with a real Buddy CSV before marking done

### Self-Improvement Loop
When corrected by Vishal:
1. Record failure in `tasks/lessons.md`
2. Identify root cause (not symptoms)
3. Write prevention rule
4. Apply in all subsequent runs

---

## DEVLOG Entry — MANDATORY

Every session must end with a DEVLOG entry. Write it at the **top** of `tasks/DEVLOG.md`.

```markdown
## [YYYY-MM-DD] [Agent] — [one-line summary]

**Mode:** Strategic | Builder | Analytical | Execution | Mixed

**Did:**
- specific completed items (file names, functions, decisions)

**State:** is it working, broken, partial?
**Decided:** architectural/approach decisions and reasoning
**Blocked / Next:** what is stopping progress, or what's next
**Modified:** exhaustive list of files changed
```

If Vishal ends the session before you write it — write it anyway as your final message.

---

## Completion Checklist

- [ ] Mode selected and stated
- [ ] Changes scoped to minimum necessary
- [ ] `node --check` passed (prototype) or `pnpm build` passed (production)
- [ ] Panel balance check passed (prototype HTML edits)
- [ ] Results documented in `tasks/todo.md`
- [ ] `tasks/lessons.md` updated if corrected
- [ ] **DEVLOG entry written**

---

## Kickoff Prompt (use to start any new session)

```
Project: Finance Buddy
Path: ~/icloud/dev-shared/projects/finance-buddy/
Goal: <clear desired outcome>
Constraints: <time / scope / tech>
Definition of Done:
- <criterion 1>
- <criterion 2>

Mode: <Strategic | Builder | Analytical | Execution | Mixed>
Complexity: <Simple micro-loop | Non-trivial full workflow>

Read: AGENTS.md, CONTEXT.md, last 5 entries of tasks/DEVLOG.md, tasks/todo.md
```
