# tasks/lessons.md
> Prevention rules learned from corrections during this project.
> Every rule has: what failed · root cause · prevention rule.

---

## L-001 — Panel nesting from wrong insertion point
**What failed:** `bld(6)` (Transactions) was nested inside `bld(5)` (Savings). When sw(6) fired, the outer `if(id===5)` was false so bld(6) never executed. Transactions tab was blank.

**Root cause:** Used `html.rfind('}\n}\n\ndone[0]=1')` to find the insertion point, but this positioned the new code inside the last Chart.js options object's closing braces, not outside bld(5).

**Prevention rule:**
- When inserting code relative to a closing `}`, always count brace depth explicitly — never rely on `rfind` for nested structures
- After inserting: search for `bld ids` and assert each appears exactly once
- After inserting: verify bld(N) is NOT nested inside bld(N-1) by checking surrounding context

---

## L-002 — Panel p6 nested inside p5 HTML (same root cause, different layer)
**What failed:** `<div id="p6">` was inserted inside `<div id="p5">`. Browser absorbed p6 into p5 (which was `display:none` when inactive), hiding Waste, Savings, and Transactions tabs entirely.

**Root cause:** Used `html.rfind('</div>\n</div>')` which found divs INSIDE p5's content, not the panel's own closing `</div>`.

**Prevention rule:**
- **Always use a brace-depth counter** (walk char by char, track `<div` = +1, `</div>` = −1, stop at 0) to find a panel's true closing tag
- After any panel insertion, run: `opens = re.findall(r'<div\b', block)` and `closes = re.findall(r'</div>', block)` for ALL panels — assert `len(opens) == len(closes)` for each

---

## L-003 — Emoji replacement eats closing tag `<`
**What failed:** Icon replacement loop used `html.replace('>Text<', '>Icon Text')` which consumed the `<` from `</span>` or `</div>`, producing malformed `>/span>` HTML. This created 3 unclosed `<div>` tags in Income panel (p3), swallowing Waste/Savings/Transactions.

**Root cause:** The `<` in the search pattern `>Text<` is the start of a closing tag (`</span>`). The replacement dropped it, turning `</span>` into `/span>` (invalid).

**Prevention rule:**
- ALWAYS preserve the trailing `<` in icon replacements: `html.replace('>Text<', '>Icon Text<')` — the `<` belongs to the NEXT tag
- After any batch icon replacement: run `re.findall(r'&#\d+;[^<>]{0,60}/[a-z]+>', html)` — any match = broken closing tag
- Run panel balance check after ANY icon replacement pass

---

## L-004 — `const tc/gc` blocks theme switching
**What failed:** Theme toggle updated `tc` and `gc` but they were declared `const` — JavaScript silently failed the reassignment, charts re-rendered with wrong colors (dark grid lines in light mode).

**Root cause:** `const` in JS blocks reassignment; no error thrown in non-strict mode.

**Prevention rule:**
- `tc` and `gc` (chart color variables) must be declared `let`, not `const`
- Whenever adding a theme toggle: grep for `const tc=` and `const gc=` and change to `let`

---

## L-005 — Yearly charts builder never reached bld(1)
**What failed:** Yearly charts (c-yr, c-st) didn't render in Trends tab despite being added. They were defined but the bld(1) function's closing brace was found at a different offset than expected.

**Root cause:** String replacement `html.replace(old_bld1_close, new)` failed silently because the whitespace/content didn't match exactly. The builder code was never injected.

**Prevention rule:**
- After ANY bld() injection: verify by searching for the key identifier (e.g., `ci["yr"]=new Chart`) in the actual bld(N) slice: `html[bld_N_start:bld_N_end]`
- Use direct byte-position insertion (find exact offset, slice + concatenate) when string replacement is unreliable
- Always run `node --check` and look for the injected code in the right bld() slice

---

## L-006 — Duplicate bld(4) blocks from failed reindex
**What failed:** After merging Yearly into Trends, old `if(id===4){ ci["yr"]=...` block remained alongside the new Waste `if(id===4){ var WASTE=...`. Two bld(4) blocks = wrong tab rendering.

**Root cause:** Reindexing only moved new content (Waste 5→4) but didn't remove the old Yearly bld(4) block.

**Prevention rule:**
- After any tab reindex: search for ALL `if(id===N)` occurrences and assert each N appears exactly once
- When merging tab content: explicitly search for and DELETE the old bld() block after the content has been moved

---

## L-007 — Category field is parts[2] not parts[1] in CSV
**What failed:** Python verification script used `parts[1]` (Head Category) as the category, giving wrong counts/totals for category-level searches.

**Root cause:** CSV columns are `Date;Head category;Category;Note;Amount;Currency` — Head category is parts[1], actual Category is parts[2].

**Prevention rule:**
- Finance Buddy CSV: `Category = parts[2]`, `Head Category = parts[1]`, `Note = parts[3]`, `Amount = parts[4]`
- TX_RAW stores `t[1] = Category` (parts[2]) — always verify against this when debugging search/filter logic
