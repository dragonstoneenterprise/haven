---
created: 2026-07-09
updated: 2026-07-11
stage: projects
---

# floatless — BFMR fulfillment tracker

BFMR (buy-for-me retail) fulfillment tracker for the reselling business. Tracks orders, matches them against credit card statements, surfaces real profit/cashback/payout. Two LLCs with separate books: **Dragonstone Enterprises** and **Cryptic Dragon**.

Stack: React PWA + Supabase + Vercel.
Repo: github.com/dragonstoneenterprise/floatless · Live: floatless-15d4.vercel.app · Supabase: plzneojxlqleaeprbygk

## Current State

- Dashboard live: Bank $261.59 (Cryptic Dragon alone; Dragonstone $300 separate = $561.59 combined), Safe to Spend $3,219.18, Locked In $39.17, Profitable Credit $700.
- Reconcile: 75/77 (97%) — the 2 remaining are iPad 9 orders with no purchase date.
- Active orders: 0 (all dead deals cancelled).
- Statement rows in Supabase: Amex 5008 (125), 2004 (64), Chase 5899 (95), Citi 4397 (25).
- Edge functions: tracker-import v12, deal-scanner v5. Latest deploy clean (TypeScript fix for welcome bonus test fixtures unblocked 5 queued commits).
- Tier 3 Tasks 1 & 2 done (nav cleanup 8→5 chips; orders pagination client-side 25/page). **Not committed.** Task 0 (card reconciliation) and Task 3 (welcome bonus fields) blocked.
- Multi-bank-account plan written, not started.

### Cards
| Card | % | Limit | LLC | Orders |
|---|---|---|---|---|
| Chase Amazon (→ rename to Amex 5008) | 5% | $700 | Cryptic Dragon | 102 |
| Amazon Visa 5899 | 3% | $3,000 | Cryptic Dragon | 2 |
| Citi Double Cash 4397 | 2% | $1,840 | Cryptic Dragon | 0 |
| Coinbase 4173 | 2% | $4,000 | Dragonstone | 0 |
| Amazon prime (orphan) | 5% | $700 | none | 0 — keep archived |

`card_statements` also has 64 rows for **2004** and 3 for **9413**; neither has an active card row yet.

### Blocker: Amex 2004
Assumed 2004 is 5008's renewed number, but the data contradicts it: 2004 and 5008 charged concurrently every month for 18 months, 2004 busier in early 2025, and each has independent payments (2004: 18 payments totaling $3,938.74) — looks like two separate accounts. One order matches only a 2004 charge; deleting those 64 rows drops reconcile 75/77 → 74/77.

Supporting evidence from earlier records: 5008 replaced **4001** (same account, both 5%), not 2004 — and 2004 was rated 2%, a different cashback tier than 5008's 5%. Both points favor 2004 being a separate account.

**Resolution:** open the Amex app — is 2004 an open account with its own balance/statement? 2 minutes, settles it.

Safe non-destructive fix regardless of answer (archived = hidden from UI, excluded from Profitable Credit, but the 64 statement rows still resolve; reversible in one line):

```sql
insert into cards (user_id, bfmr_account_id, name, last4, credit_limit_cents, default_cashback_pct, archived_at)
select user_id, bfmr_account_id, 'Amex ****2004', '2004', 0, 2.00, now()
from cards where last4 = '5008';
```

## Decisions

Architecture invariants:
- Money as integer cents (dollars only at render).
- `qty` authoritative from BFMR.
- Never overwrite user-set fields (card/notes/buyer) on import.
- Never hard-delete a card — archive via `archived_at`.
- All data scoped per `bfmr_account_id`.
- Statements & Amazon orders persist to Supabase, never re-upload.

Other decisions:
- **Profitable Credit $700 is correct** — only one active 5%+ card (5008, $700 limit, $0 balance). The earlier "$1,400" was wrong: Amazon prime misread as a counted duplicate; it's orphaned (`bfmr_account_id = NULL`, 0 orders) and invisible to the app. *Lesson: always check `bfmr_account_id` + order counts before calling a row a duplicate.*
- **Orders pagination is client-side**, not server `.range()` — filters/badges/summaries compute over the full 104-order set; server pagination would scope them to 25 visible rows and break everything. Perf case is thin at 104 orders.
- **Keep Amazon prime orphaned + archived** — assigning it an LLC makes it visible and doubles Profitable Credit to a phantom $1,400.
- **Cashback stored per-order** (`expected_cashback_pct/cents`), not looked up from card, so card changes don't alter historical math.
- **Default account: Cryptic Dragon** (most-used). `is_default` was false on both; corrected to true on Cryptic Dragon.
- **Multi-bank plan uses expand/contract**: new `bank_accounts` table (FK to `bfmr_account_id`) → backfill → cutover → verify → drop old columns. Hard rule: **never sum bank balances across LLCs** (would show $561.59 as one company's cash).

### Schema
`profiles` (user profile) · `orders` (all BFMR orders) · `cards` (credit cards + cashback rates) · `scanned_deals` (deal scanner results) · `bfmr_accounts` (multi-account credentials + bank balance) · `card_statements` (all card CSV transactions, persisted) · `amazon_orders` (Amazon order history, persisted; unique on `user_id + order_id + model_number`).

Orders store **per-unit** money; UI multiplies by `qty` everywhere.

### Cashback rates
| Card | Rate | Note |
|---|---|---|
| Amex 5008 | 5% | main card; replaced 4001 (same account, also 5%) |
| Chase Visa 5899 | 3% | corrected from 5% |
| Amex 2004 | 2% | see blocker above |
| Citi 4397 | 2% | |
| US Bank 9413 | 2% | used for Amazon gift card purchases |

Key insight: gift-card orders still earn 5% because the gift cards were purchased on Amex 5008.

### Reconciliation — 5 matching methods, priority order
1. Amazon order ID — exact match
2. TBA tracking number — proof of shipment; covers gift-card-paid orders (**the key unlock: 77% → 97% verified**)
3. Card charge amount ±$1 within 14 days symmetric
4. Item model number — matches Amazon CSV
5. Group charge — sums same-day orders vs bundled card charge ±$2

Cost correction applied: Jun 2026 Echo Dots were $29.99 in DB, actual $34.99 — fixed.

### History (done, kept for context)
- 97 duplicate ghost orders from early imports deleted via SQL.
- Dead orders cancelled (iPad 9 Silver/Space Gray, Watch SE 2024 44mm Midnight, Watch Series 10 — no tracking, no `paid_at`); they were inflating safe-to-spend by ~$2,051.
- Order fields added over time: `amazon_order_id` (tracker-import v11, shown on OrderCard), `tracking_number` (v12, shown on Get Paid page), `verified` boolean (reconcile page). Edit unlocked on all statuses to fix card assignments on historical orders.
- Universal CSV parser auto-detects Amex/Chase/Citi/any bank; keyword + date-range filters on Amex Import page; multi-card balance sum across all cards.

## Next Steps

1. Check Amex 2004 — open or closed? Real limit? (settles Task 0 migration)
2. Commit both plan files to `plans/`; delete `PLAN-tier3-ux-polish.md` v1 so the stale version with the wrong $1,400 can't be picked up.
3. Run Task 0 with real 2004 status + 9413 real limit ($500 / 2%, verify), using a verification query that can actually fail (each `card_statements.last4` maps to exactly one active card).
4. Run Tier 3 Task 3 (welcome bonus fields; assert Profitable Credit = $700).
5. Run multi-bank plan Tasks 1–5.
6. Backlog: sync `current_balance_cents` (all $0 today, caveat needed in UI); Tier 1 safe-to-spend verification; Tier 2 bundle matching / 2nd-account scan / deal-scanner card view; Tier 4 ErrorBoundary / `getSession`→`getUser` / real cycle dates.

### Rename track (separate)
"Floatless" reads weak (negative framing, jargon). Dictionary finance words taken: Reckon (ASX/Accounting), Tally (TallyPrime), Ingot (INGOT Brokers); descriptive terms too generic → go with a coined word. Leading candidates:
- **Hoardly** — Dragonstone-native (dragon guards hoard), memorable, strong SaaS branding
- **Recoupa** — from "recoup," states the job clearly

Both need USPTO TESS search (software class) + domain check before committing.
