# Design request: Expense app — payment-method as the unit; invoices as expand-to-itemize (resolving source conflation + retail-receipt double-count)

**Status:** open
**Filed:** 2026-05-22
**Author:** orchestrator (user-initiated after a session of fighting the double-count bug)
**Surface in scope:** the personal-finance expense tracker at `/home/claw/work/finance/expense` (route family `/transactions`, `/breakdown`, `/spending`). This is *not* part of the Reeloo learning apps; it's the orchestrator's own infra for tracking household spending. Same designer though, same visual system desired.

**Trigger (user, verbatim, this session):**

> we shouldn't mix payment with merchant in sources, that can double count expenses: e.g. i pay lidl groceries using swisscard, then i'll see one from lidl the other from swisscard. also how do I filter by multiple merchants: lidl + coop + migros + denner. filter by payment (chase, swisscard, zkb ...etc). or you have ideas how to present this?

---

## 1. The problem in one sentence

The DB has a column called `source` that overloads three distinct concepts — *where this row came from*, *which payment instrument the user used*, and *which retail receipt enriched this purchase* — so every Lidl/Coop/Migros purchase shows up at least twice in `/transactions` and is double-counted in any naive aggregation.

## 2. Concrete evidence of the bug

A 60-day SQL probe found 12 retail receipts (source = `lidl` / `supercard`) that have an exact date+amount mirror in a bank-statement row (source = `swisscard` / `zkb-6359` / `zkb-8550`). Examples:

| date | CHF | retail row | statement row |
|---|---:|---|---|
| 2026-05-15 | 59.85 | `supercard` "Coop Pfäffikon SZ Seedamm" | `swisscard` "Swisscard Coop" |
| 2026-04-30 | 4.99 | `lidl` "Lidl Altendorf (1 items)" | `zkb-6359` "Transaction ZKB Visa Debit ... LIDL ALTENDORF" |
| 2026-04-14 | 67.30 | `lidl` "Lidl Altendorf (35 items)" | `zkb-8550` "Transaction ZKB Visa Debit ... LIDL A..." |
| 2026-04-13 | 52.45 | `lidl` "Lidl Altendorf (32 items)" | `swisscard` "Swisscard Lidl" |
| 2026-05-15 | 25.15 | `supercard` "Jumbo" | `swisscard` "Swisscard JUMBO" |

The same problem exists at the bank-statement layer too: `chase-email` (real-time push notification) and `chase-7531` (end-of-cycle statement CSV) deliver the same Costco charge twice. The user spotted this when an "April Costco summary" showed CHF 2,186 in 30 days; reality after dedupe was CHF 1,506. The duplicated CHF 681 of statement-vs-email overlap matched a known dedupe-bug pattern.

## 3. Where the conflation comes from

The `Transaction` table has one column `source` that today carries values from three buckets:

| Bucket | Example values | What it really means |
|---|---|---|
| **Statement / cash event** (authoritative) | `chase-7531`, `chase-4685`, `swisscard`, `zkb-6359`, `zkb-8550`, `zkb`, `bcge`, `revolut`, `ibkr`, `schwab`, `hsbc-tw`, `ctbc`, `chinatrust-3751` | A real cash movement from a bank/card statement or broker activity feed |
| **Realtime notification** (duplicate of statement) | `chase-email` | Push notification from Chase Mobile that arrives ~5 days before the statement CSV — same purchase, double-recorded |
| **Retail receipt** (enrichment, not cash) | `lidl`, `supercard`, `migros`, `costco` | Itemised receipt from the retailer app; mirrors a card/bank entry. Adds *what was bought*, not *that money moved* |

A `Lidl 70.55 CHF` purchase on 2026-04-13 paid with Swisscard therefore lives in DB as:

- Row A: `source=swisscard` `description="Swisscard Lidl"` `amount=-70.55` — the cash event
- Row B: `source=chase-email` (if Chase was used) or no second row (if Swisscard) — the realtime ping
- Row C: `source=lidl` `description="Lidl Altendorf (49 items)"` `amount=70.55` `reference=lidl-04000154120260413...` — the itemised receipt

`/transactions` shows all three. `/breakdown` sums all three. The user genuinely paid CHF 70.55 once.

## 4. What I've shipped this session as the band-aid

1. Added `excludeRetailReceipts` param to `/api/transactions` (default `true`) that filters out `source IN (lidl,supercard,migros,costco)`. Verified the const + where-clause; rebuild + restart done.
2. Added a `Custom` period option + two `<input type="date">` controls to `/transactions` so the user can do `[2026-04-22 → 2026-05-22]` instead of preset windows only.
3. Bulk-recategorized 426 HSBC-TW inter-account FX leg rows from `Uncategorized` to `internal_transfer` (persisted into `config/category-overrides.json`), bringing the 6-month Uncat from CHF 21,793 → 16,853 (−23 %).
4. Built a `/api/import/retail` endpoint so the Rays-iMac receipt-scrape daemon can POST CSVs directly instead of dropping them on disk and hoping someone runs `import-lidl.ts`.

The band-aid is enough to stop the double-count *for now*, but it leaves the right thing — a real "Transaction view" — unbuilt. Hence this design request.

## 5. The model (user's framing — confirmed mid-thread)

**Transactions are tracked by payment method. Invoices expand a transaction into items.**

- **Transaction** = one cash event on a payment instrument's ledger. The unit of accounting. Source of truth for "money moved": Swisscard CSV, ZKB e-Bill, Chase statement, BCGE CAMT, etc.
- **Invoice** (= retail receipt) = the itemised breakdown of *what a single transaction bought*. Lidl receipt for CHF 100 = 5× eggs + 3× hams + 2× milk. The invoice does NOT represent additional money; it explains an existing transaction.
- **Mapping** is many-to-one Invoice → Transaction (usually 1:1; can be N:1 if a single statement leg actually settles multiple receipts — Costco gas + warehouse split is the only N:1 case I've seen).
- **The UI affordance** for the user: every transaction row gets an expand arrow `▸` iff a linked invoice exists. Tap to expand inline → item-by-item list. No expand arrow = no invoice; row stays unexpandable.

What the schema needs:

- A new `Invoice` table (or reuse current retail rows): `Invoice(id, source: 'lidl'|'supercard'|'migros'|'costco', date, merchant, total_chf, items: jsonb, raw_receipt: text/blob, linked_tx_id INT NULL)`
- `Invoice.linked_tx_id` = FK into `Transaction.id` — null until the linker matches it
- Aggregations never read from `Invoice` directly. Only from `Transaction`. Invoices are pure enrichment.
- The current rows in `Transaction` with `source IN (lidl,supercard,migros,costco)` get **migrated** into the new `Invoice` table and removed from `Transaction`. After migration there is no double-count by construction — retail data isn't even in the cashflow ledger.
- `chase-email` rows that mirror a `chase-XXXX` statement row also get demoted: they belong in a `RealtimeNotification` enrichment table (same idea — early ping, not a separate cashflow), with a `linked_tx_id` back to the statement row.

What the linker decides (run nightly or on-import):

- For each newly-arrived retail Invoice, find the matching Transaction on the same payment instrument by `(date ± 1 day, amount within 1% or 0.05 CHF, merchant resolves via aliases)`. If exactly one match → set `linked_tx_id`. If 0 matches → leave NULL, surface in a "review inbox" UI. If 2+ matches → also surface for user resolution.
- Same for `chase-email` ↔ `chase-XXXX`.
- The match decision is reversible — user can unlink + relink in the UI.

## 6. The presentation question (this is what we want Claude design's eye on)

Once we have Transaction as a first-class concept, the UI should let the user:

### 6.1 Browse transactions (list view, with inline expand)

Collapsed state — one row per cash event:

```
▸ 2026-05-15  Coop Pfäffikon SZ Seedamm     59.85 CHF   Swisscard 0794    (28 items)
▸ 2026-05-12  Lidl Altendorf               119.78 CHF   ZKB Visa Debit 6359 (35 items)
  2026-05-08  CSS Krankenversicherung    1 605.00 CHF   ZKB e-Bill
  2026-05-07  Costco Caisses               813.04 CHF   Chase CSR 7531
  2026-05-07  Costco Gas (Sodano)           23.56 CHF   Chase Freedom 4685
  2026-05-05  SBB CFF FFS                  600.00 CHF   Swisscard 0794
  2026-05-05  SBB CFF FFS                  600.00 CHF   Swisscard 0794    ← confirmed legit dup
```

Rows that *have* a linked invoice get a `▸` disclosure caret and a faint item-count badge. Rows without an invoice (`CSS Kranken`, `Costco gas`, `SBB`) just sit there flat — nothing to expand. The user knows at a glance which rows are itemisable.

Expanded state — tap the caret on the Lidl row:

```
▾ 2026-05-12  Lidl Altendorf               119.78 CHF   ZKB Visa Debit 6359 (35 items)
   ┌──────────────────────────────────────────────────────────────┐
   │  Milk H-Milch 1L           × 2       3.60                    │
   │  Bananas                   1 kg      2.20                    │
   │  Brot Vollkorn             × 1       4.80                    │
   │  Käse Greyerzer 250g       × 1       9.95                    │
   │  …32 more lines…                                              │
   │  ─────────────────────────────────────                       │
   │  Subtotal                          119.78                     │
   │  Paid: CH 2026-05-12 17:42  ·  Karte ****6359                │
   │                                          [open Lidl receipt ↗] │
   └──────────────────────────────────────────────────────────────┘
```

The expansion is **inline**, not a modal — keeps the user in the list, can keep scrolling with item-context still visible.

**Open design question for Claude:** how to express "Items available" subtly without making the row visually noisy? An icon-only chip? A faint count badge `35 items` on the right? An expand-arrow that's always visible (per the mock above) vs only on hover? Need it readable in a 100-row dense list, on mobile too.

### 6.2 Filter chips (what we'd want)

```
[ 🔍 search description     ]  [ Period: Custom 2026-04-22 → 2026-05-22 ]
[ Merchant: Lidl, Coop, Migros ×]   [ Paid with: Swisscard, ZKB ×]
[ Category: Groceries ×]            [ Currency: CHF ×]
[ Show retail-receipt rows separately ☐ ]   (debug toggle; default off)
```

Multi-select chips with type-ahead. **Merchant** chip resolves to canonical names via aliases (so picking "Lidl" matches `lidl-XXX` retail rows + statement rows whose description matches Lidl-alias). **Paid with** chip lists the user's payment instruments (mapped from underlying `source` values, presented in human-friendly names with last-4-digit suffix where relevant).

**Open design question:** chip pickers should be searchable since the user has ~25 distinct merchants in steady-state, but only ~6 are common. Visual treatment when there are >3 selected: do we collapse to a count badge (`Merchant: 5 selected`) with expand-on-click, or keep them all inline?

### 6.3 Group-by views

User should be able to flip a single control between four views:

1. **By date** (default — chronological list as above)
2. **By merchant** — collapsed list of merchants with totals; expand to see purchases under each
3. **By payment instrument** — see "How much did I put on Swisscard 0794 this period?" — collapsed → expand
4. **By category** — same model as `/breakdown` today, but with the Transaction row as the leaf

**Open design question:** is one of these the dominant entry point that should be the *home* of the expense app, with the others as alternate lenses? Today `/transactions` is chronological by default, but I suspect "By merchant in last 30 days" is the more frequent question the user actually asks.

### 6.4 The expand affordance (replacing "drawer" with inline)

Inline expansion is the primary pattern (see 6.1). The drawer is only a fallback when the row is too dense on mobile or the item list is too long to inline comfortably (Costco receipts can have 60+ items).

On mobile: inline expand if the linked invoice has <15 items, otherwise an icon `📋` opens a bottom-sheet.
On desktop: always inline.

Drawer / bottom-sheet content (when used):

```
Lidl Altendorf — 2026-04-13 — 70.55 CHF — paid Swisscard 0794
─────────────────────────────────────────────────────
  Milk H-Milch 1L × 2                  3.60
  Bananas 1kg                          2.20
  Croissant butter × 4                 4.80
  … 49 lines total …
─────────────────────────────────────────────────────
  [download original receipt PDF ↗]   [see all Lidl transactions →]
```

PDF availability is per-source (Coop: yes; Lidl: no; Migros: varies). When unavailable, hide the button — don't show it greyed.

### 6.5 Summary band at the top

```
Last 30 days — CHF 10 076 spent across 111 purchases
  Groceries 4 666 (46%)  ·  Rent 2 430 (24%)  ·  Insurance 1 605 (16%)  ·  Transport 1 489 (15%)
  Top merchants: Costco 1 506 · Lidl 1 188 · Coop 480 · CSS 1 605
  vs same week last year: ↓ 23%
```

A live-updating band that re-computes whenever filters change. Treemap or stacked bar for the category share? Or text-only with sparkline? Today's `/spending` page tries to do annual roll-ups; the band here is the *period-local* counterpart.

### 6.6 The "two SBB CHF 600 same day" question

The linker (Invoice → Transaction) is asymmetric with the dedupe question. Two statement-source rows on the same date with the same amount are *both real transactions* by default — never collapsed. The linker only collapses pairs where one is statement-source and the other is enrichment (retail invoice OR realtime notification). So `swisscard CHF 600 SBB` + `swisscard CHF 600 SBB` stays as two rows; `swisscard CHF 70.55 Lidl` + `lidl invoice CHF 70.55` becomes one row (with expand affordance).

That said: if the user *believes* two statement rows are accidentally double-recorded (unusual but happens with Chase 2-day-pending-then-posted artefacts), they should be able to **manually merge** two transaction IDs. And inversely: if a single statement row actually paid two real-world purchases (rare — usually only Costco split charges), they should be able to **manually split** one row.

**Open design question:** how to expose "merge these two rows" and "split this row" without bloating the toolbar? Long-press multi-select on mobile? Cmd-click on desktop? A row-level kebab menu with `Merge with…` / `Split into…` options?

## 7. What's NOT in scope for this design pass

- The Transaction linker algorithm itself (that's backend work)
- Schema migrations (backend work)
- Multi-currency display logic (already solved in `/lib/historical-display-currency.ts`)
- Auth / sharing (single-user app)
- Mobile native app (web-only for now; works on mobile-web)

## 8. Reference material in the repo

- Existing transactions UI: `~/work/finance/expense/src/app/transactions/page.tsx`
- Existing breakdown UI: `~/work/finance/expense/src/app/breakdown/page.tsx` (best current state of "category → merchant drill" with date pickers)
- Existing spending annual roll-up: `~/work/finance/expense/src/app/spending/page.tsx`
- Classifier (ML category suggestions) UI: `~/work/finance/expense/src/app/classifier/page.tsx`
- Schema: `~/work/finance/expense/prisma/schema.prisma` (look at `Transaction` model — note existing `parent_transaction_id`, `is_split_parent` columns, used by another linker for split transactions but not for purchase-grouping)
- Counterparty aliases: `~/work/finance/expense/config/counterparty-aliases.json` (currently has 2 entries — sparse; design should assume this grows to ~100 once the Transaction linker is wired)
- Category overrides: `~/work/finance/expense/config/category-overrides.json` (driven by the `/classifier` UI)
- Today's visual system: same Tailwind + shadcn primitives + `var(--text-primary)` / `var(--card-bg)` token system that Reeloo apps use. Themes: vanilla, cobalt-dark, high-contrast (same lock as v2/06 §6.2).

## 9. What I want back from Claude design

1. Mockups for the four views (date / merchant / payment / category) at production density — desktop and mobile.
2. The filter-chip bar treatment — multi-select chips, type-ahead, overflow behaviour.
3. The Transaction-row component: how the `📋 items` affordance is shown, how the row reacts to hover/tap, how the category badge and payment-instrument badge sit.
4. The item drawer (mobile bottom-sheet + desktop side-panel) — including the "see all <merchant> purchases" link.
5. The summary band — pick a viz, justify it.
6. The bulk-edit affordance for merge/split purchase IDs.
7. Empty states for: no purchases in window, no retail receipt linked, no payment instrument resolvable.
8. Error states for: linker disagreement (the system thinks two rows are the same purchase but the amounts mismatch by > tolerance — should we surface a "review this" inbox?).

Light-touch is fine — these are all single-user views and the user's already past the "is this useful?" stage. The bar is "would I rather use this than my Excel sheet?", not "is this beautiful enough to put on Dribbble".
