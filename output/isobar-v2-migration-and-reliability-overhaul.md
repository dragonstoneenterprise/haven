---
created: 2026-07-11
updated: 2026-07-11
stage: output
harvested: true
harvested_date: 2026-07-11
source_project: "[[isobar]]"
---

# Isobar: Kalshi V2 migration + reliability overhaul

Shipped across three engagements (Jun 24 – Jul 10, 2026). Kalshi deprecated the legacy order endpoint (410 Gone), which had caused 2+ weeks of silent zero fills. Fixed the migration, then used the outage as a forcing function to audit the whole bot for silent-failure patterns.

## What shipped
- Migrated `place_order` to Kalshi's V2 `POST /portfolio/events/orders`, including correct YES-relative side/price mapping and IOC (not GTC) semantics.
- Rewrote win/loss determination to read Kalshi's `/portfolio/settlements` ledger only — never guess from `get_market()`.
- Rewrote P&L to compute live from Kalshi's real balance every call — deleted every hand-maintained accumulator (`bot_state.total_pnl`, etc.), all of which had silently drifted from reality.
- Found and fixed a coordinate bug: signal generation and settlement fallback both used city-center coordinates instead of the airport stations Kalshi actually settles against (LA was off by ~17°F for its entire 81-day history).
- Rebuilt the launchd daemon (was misconfigured as a once-daily job, not a persistent process) and hardened exit codes so any non-`botstop` shutdown always triggers a restart.
- Added a 3-state (PASS/DEGRADED/FAIL) daily health check that distinguishes real failures from transient network issues.
- Net result: true realized P&L reconciled from multiple false headline numbers (+$576.88, +$398, +$108.63 — all bugs) down to the honest figure, −$5.85.

## Why it matters beyond this bot
Every fix here was a variant of the same failure mode: **something silently drifted from ground truth and nothing ever re-checked it.** Ghost trades, a hardcoded sync constant, an `lru_cache`d failure, an unfiltered paper-trade sum, a `sys.exit()` that didn't actually exit — seven distinct bugs, one root pattern. That pattern will recur in any bot that touches money or external state, including gold-bot/Assay.

Full detail: [[isobar]]
