---
created: 2026-07-11
updated: 2026-07-11
stage: wiki
source: "[[isobar-v2-migration-and-reliability-overhaul]]"
---

# Source of truth, not local accumulators

Any time a system tracks a running total (P&L, balance, win count) in its own database instead of computing it fresh from the external system of record, that total will eventually drift — silently, with no error, no crash, nothing to notice.

Isobar hit this same bug in five different forms:
- A hand-incremented `total_pnl` counter, incremented on trade placement *and* settlement, never decremented on voided trades — drifted to a number matching no real combination of trades.
- A "sync with reality" button that actually added a hardcoded constant instead of reading the real balance — the dominant cause of a fake +$576.88 headline.
- Paper (fake-money) trades summed into the same accumulator as real trades.
- A cached credential-check result that stayed "failed" forever after one transient network blip, because success was never separately cached from failure.
- Win/loss read from a fast-but-sometimes-premature API field instead of the exchange's authoritative settlement ledger — flipped 31 of 149 historical trades.

## The rule
- Compute derived numbers (balance, P&L, status) live from the authoritative external source on every read. Never maintain a parallel counter that's supposed to match it.
- If you must cache something that can fail, cache the success — never cache a failure indefinitely.
- When two records disagree, the external system of record wins, not your local database.

## Where else this applies
Any bot or app that mirrors state from a bank, exchange, or third-party API: don't keep your own running total of "what should be true" — ask the source.
