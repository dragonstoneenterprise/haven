---
created: 2026-07-09
updated: 2026-07-11
stage: projects
---

# isobar — Kalshi weather trading bot

Python/FastAPI bot ("Isobar" — chosen name, not yet applied; code still branded "Neon Empire / WX Console v5", agent-002-weather) that autonomously trades Kalshi daily high/low temperature markets (KXHIGH*/KXLOW*) across 12 US cities. Strategy: GFS ensemble (31-member) vs ECMWF (51-member) via Open-Meteo — trade only when both models agree within 3.5°F, then buy the favored side when edge ≥ 8%. Runs locally on the MacBook Air via launchd daemon.

Stack: Python 3.11, FastAPI/uvicorn (http://127.0.0.1:8000), SQLite (`tradingbot.db`), APScheduler, Bitwarden Secrets Manager, Telegram alerts, Kalshi API (RSA-PSS signed, base `https://api.elections.kalshi.com/trade-api/v2`), Open-Meteo.
Local path: `~/neon-empire/agents/agent-002-weather` · no public repo · Kalshi account mushudragon01, Basic API tier. Real UI is `index.html` (served at root) — `dashboard.html` is a decoy/unused duplicate.
Backups: `~/bot-insurance/20260705/` (code tarball, DB snapshot, plists, zshrc) + `~/bot-insurance/RECOVERY.md` (no-AI-needed recovery steps).

## Current State

*(as of Jul 10, 2026 chat — newest archived)*

- **Bot running, launchd-managed, self-healing** (RunAtLoad, KeepAlive.SuccessfulExit=false, ThrottleInterval 30; verified by kill-and-respawn test, new PID in ~18–35s). Nightly stop plist deleted — 24/7 daemon now.
- **True realized P&L, all-time, full Kalshi ledger (175 real settlements): −$5.82** (refined −$5.85 pending orphan-backfill tie-out; 3¢ is rounding noise). This is the honest number after multiple fake numbers were corrected (+$576.88, +$398, +$108.63 — all accumulator artifacts).
- **Orphan backfill (10 rows, −$13.65) approved but completion NOT confirmed — verify it ran.**
- `healthcheck` zsh alias runs `health_check_daily.py`: 4 checks (process, scan-gap, Kalshi reconciliation, P&L sanity), 3-state verdict PASS/DEGRADED/FAIL (exit 0/2/1).
- **LA coordinate bug fixed Jul 10** — all 52 LA trades over the bot's entire 81-day history had signals generated from downtown-LA weather instead of LAX (~16.9°F average gap). Zero post-fix live data yet.
- **NYC unresolved:** `nws_station: "KNYC"` is not a real ICAO code; unknown which station Kalshi settles NYC against (Central Park? LGA? JFK?). Deliberately not guessed — flagged for investigation.
- Position cap `max_open_positions=5`, live-mode only (paper excluded from the gate — scheduler was always correct; only the dashboard display blended them, now fixed). ~4% bankroll exposure at cap. On record: leave at 5 — no demonstrated durable edge to justify more risk.
- Cities live: miami, los_angeles, denver, austin, atlanta, chicago, dallas, phoenix, seattle. Paper: nyc, minneapolis, boston. *(Resolves the Jul 3 austin contradiction: austin is live.)*
- **auto_tune OFF.** Framework exists (promotion_min_trades 10, min_win_rate 0.6, min_brier_improvement 0.02) but sample far too small. Standing rule: 50+ trades minimum per parameter change + human approval before ever enabling.
- Naming decided, not executed: weather bot → **Isobar**, gold bot → **Assay**. No umbrella/company name finalized.

## Decisions

### Signal gate — the core edge (FROZEN; change only with audit + human approval)
- GFS/ECMWF divergence ≤3.5°F · min edge 8% · favorable-side model prob ≥0.65 (raw) · entry ∈ [0.03, 0.45] · buffer ≥1.2°F from threshold · yes_price ∈ [0.05, 0.95] · 4pm-ET same-day cutoff · MAX_TRADES_PER_SCAN=5 · MAX_TOTAL_PENDING=5 · $500 weather allocation cap. Sizing: Kelly 25% clamped by max_trade_size 15 → ~$8 flat.
- Why 0.65: audited winners had model_prob 0.74–0.95, losers 0.13–0.29. Why max 0.45: expensive NO trades (>0.55) lost money even at 55% win rate (−$21 bucket).
- **Ensemble-unanimity filter (Gate A) audited twice and kept.** Full counterfactual (Jul): 574 blocked signals over 26 days would have won only **5.2%**, net −$165.47, losing in every city and probability bucket, vs 40.5% on trades actually taken. Overturned the standing "filter is backwards" hypothesis with data. Gate B (mirror-image case) got the same direction-aware fix for consistency — financially immaterial (n=4 in 26 days).
- `learning_nightly` confirmed safe (seeds promotions only, owner approves via API). auto_tune stays OFF per above.
- **Do not tune strategy on small samples** — win rate alone is the wrong dial (buying at $0.90 wins 90% and loses money).

### Naming & structure
- **Isobar** (weather pressure lines) and **Assay** (gold purity testing) chosen — both clean in search, real domain terms, easy to say. Rejected for the gold bot: Aurum, Midas (both scam-adjacent in existing products), Solidus, Croesus (collide with funded finance companies). Rejected the unified "Wyrm-" prefix family (Wyrmhoard/Wyrmwind) as "too cheesy" despite the stronger Dragonstone narrative. No umbrella name finalized.
- **Two bots kept architecturally separate** (Kalshi vs Bitget, different asset classes, different failure domains) — brand-level unification only, never code/infra.

### Kalshi integration
- **`place_order` on V2 `POST /portfolio/events/orders`** — legacy endpoint 410 Gone (deprecated May 2026, killed late June); was THE root cause of 2+ weeks of zero fills. V2 mapping is YES-relative: `yes`→`bid` @ price/100, `no`→`ask` @ (100−yes_price)/100 — getting this wrong = instant bad fills.
- **IOC, not GTC** — no fill-polling/resting-order cleanup exists; GTC could rest unfilled → inverted ghost. Deliberate.
- **Fill verification:** fill_count==0 → void row; partial → size=fill_count; missing fill_count → KEEP + ERROR (never auto-void on ambiguous data).
- **Win/loss from Kalshi's `/portfolio/settlements` ledger only** — never `get_market()['result']` (returns premature values). No settlement record → return unresolved, retry next cycle. Never guess.
- **P&L computed live from Kalshi** — `/api/overview` pnl_total = (Kalshi balance + portfolio_value) − initial_bankroll, fresh every call. **Standing rule: never reintroduce a hand-maintained P&L/trade-count accumulator anywhere** — every one found (bot_state.total_pnl/total_trades/winning_trades) had silently drifted with no self-correction.
- Nightly reconciliation safeguard `check_for_unlogged_kalshi_settlements` (Telegram-alerted) — orphan settlements can't silently recur.

### Operations & resilience
- **launchd daemon, correctly configured** — original plist had KeepAlive:false + StartCalendarInterval (a once-daily job, not a daemon); replaced Jul with RunAtLoad + KeepAlive.SuccessfulExit=false + ThrottleInterval 30, verified by kill test.
- **Nightly stop plist deleted entirely** (not rescheduled) — pkill at 01:00 was pointless under the new config and had caused multi-hour/multi-day silent outages ≥3 times. Bot is 24/7 now that it places real trades.
- **Any shutdown not via `botstop` exits nonzero** so launchd always restarts — graceful SIGTERM (exit 0) is invisible to KeepAlive.SuccessfulExit=false; fix had to be at exit-code level.
- **Full backup taken before letting an agent touch the live daemon** (`~/bot-insurance/`) — deliberate risk mitigation with real money in open positions.
- **Health check is 3-state** (PASS/DEGRADED/FAIL; UNVERIFIED via `_is_network_error()` exception classification) — a script that fails loudly on transient network issues (33% packet loss to Bitwarden observed) trains the user to ignore it.
- **Credentials: retry-with-backoff, deliberately NO Keychain fallback for Kalshi** — judged sufficient for brief BWS outages; a judgment call, not tested against sustained outage.
- **Rejected Claude Code as unattended self-healer** — it only acts within open sessions; real self-healing lives in the bot's own code (retry, watchdog, exit-code hardening) + a scheduled daily Claude Code health-check session as a secondary layer. Fuller Sentinel/Medic/Surgeon/Warden architecture designed (Jul 3), not built.
- **Working rule (user-established): explain discrepancies to ground truth before "fixing" them** — never patch a wrong number without knowing why it's wrong. This rule found the $154 win/loss-flip bug, the $58.82-vs-−$5.82 gap, and the LA coordinate bug.
- **Not selling the bot** — alpha decay on thin markets.
- Dashboard exposes safe knobs only; signal-gate params never in UI/settings.json. Bitwarden over .env, secrets never plaintext.

### Corrections & lessons (chronological-ish)
- **Ghost trades (the founding bug):** DB row written before Kalshi order; rejections left fake open trades blocking real ones. Fixed pre-commit; 20 ghost rows later voided (not deleted — audit trail).
- **"410s = market closed" — wrong:** endpoint deprecation. *Uniform failure across all cities = systemic.*
- **Date-filter off-by-one:** fixing infinite 410 retry loops with `<= today` excluded today's open markets — a full day (Jun 30) of zero trades. *Verify date fixes against a live scan, not just "errors stopped."*
- **Paper cities blocking live trading (costliest bug):** paper NYC/Austin won the signal-ranking comparison and starved live cities ~2 weeks with zero errors shown. Fixed by splitting live_actionable/paper_actionable pools. *"The bot always feels stuck" is worth investigating immediately — zero visible errors ≠ doing anything real.*
- **`/api/sync-balance` hardcoded `total_pnl = balance − 91.0`** — stale constant unrelated to the real bankroll; every button click injected ~$200+ phantom P&L. Dominant cause of the fake +$576.88. *Sync functions derive from the authoritative source, never a frozen constant.*
- **Paper trades summed into real P&L** (no mode filter in settlement accumulator): +$159.20 phantom.
- **`get_market()['result']` flipped 31 of 149 historical trades** (30 false wins, 1 false loss — $154 swing), baked in because nothing re-checked settled trades. Fixed via settlements ledger.
- **Fees never subtracted** ($9.37 total observed). Fee formula empirically verified exact: `fee = 0.07 × price × (1−price) × count`.
- **10 orphan settlements** (3 BTC, 7 weather, −$13.65) existed on Kalshi with zero local record — inverse ghost (money moved, never logged), likely crashes between order and DB write. Backfilled as synthetic rows tagged `settled_via='backfilled_orphan_settlement'` (completion unconfirmed) + nightly safeguard added.
- **`_bw_client()` `@lru_cache` cached a connection failure permanently** — one startup blip blinded the bot to all Kalshi credentials for the process's life. Actual root cause of the Jul 7 "0 markets found" incident. *Cache success, not attempts.*
- **broker=FAIL mystery solved:** `_check_broker()` called a nonexistent `get_secret` function; ImportError silently swallowed → permanent false FAIL. Fixed; should report broker=OK now.
- **`sys.exit()` inside an APScheduler coroutine doesn't kill the process** — SystemExit caught as job exception. Watchdog fired correctly but never killed anything until switched to `os._exit()`.
- **Nightly stop plist caused ≥3 silent outages** (log mtime gaps: Jul 5 03:33 → Jul 7; Jul 7 23:59 → Jul 8 05:03). One death at 00:00:01 vs the 01:00 schedule was never explained — open forensic question if it recurs.
- **`/api/stats` fed the real UI from drifted accumulators** (total_pnl $108.63, 186W/34L — matching no real combination; DB had 245 rows, 90 real wins). Found only because the user refused "the dashboard is just wrong, fix the display" and demanded an explanation.
- **Coordinate bug (biggest lesson):** `CITY_CONFIG` used city-center coordinates, but Kalshi settles against **airport** NWS stations — and the same dict fed both the observation-fallback settlement path AND live signal generation (`fetch_ensemble_forecast`/`fetch_ecmwf_forecast`). LA worst: downtown vs LAX, ~16.9°F gap, all 81 days of LA signals wrong. 12 trades wrongly settled via fallback (7 LAX, 4 MIA, 1 CHI); 6 of 12 flipped win→loss on correction. Fixed for 9 cities; NYC left unfixed deliberately (KNYC not real, station unknown). *Always check whether a suspect data source is shared by other code paths before scoping a fix narrowly.*
- **`pgrep -f "python run.py"` lies** (Homebrew binary is capital-P `Python`) — use `lsof -ti:8000`, but note a Brave helper process has coincidentally shared that port number in listings; confirm with `ps aux`. `botstatus` right after `botstart` reads STOPPED (registration lag).
- **Kalshi app History scroll ≠ truth; bot DB accumulators ≠ truth.** The Kalshi ledger is the only source of truth.

### Reference
- **DB (`tradingbot.db`):** `trades` (id, signal_id, market_ticker, platform, event_slug, market_type, direction, entry_price, size, timestamp, settled, settlement_time, settlement_value, result, pnl, model_probability, market_price_at_entry, edge_at_entry, settled_via, mode) · `bot_state` (accumulator fields deprecated/unreliable — do not read) · `health_checks` (timestamp, per-check statuses, overall).
- **Initial bankroll (canonical): $299.38** (`settings.INITIAL_BANKROLL`). NOT $195.92 / $282.43 / $278.78 — invalid artifacts from broken accumulators.
- **Formulas:** fee = 0.07 × price × (1−price) × count (verified exact). True P&L = (Kalshi balance + portfolio_value) − initial_bankroll. Realized-only = Σ pnl over settled, non-void, non-paper trades, verified ticker-by-ticker against `/portfolio/settlements`.
- **Kalshi:** amounts in CENTS; ticker `KXHIGH<CITY>-<YY><MON><DD>-{B<x>.5 band | T<x> threshold}`; RSA-PSS signing in `backend/data/kalshi_client.py`.
- **Secrets:** `kalshi_api_key`, `kalshi_api_secret`, `telegram_token`, `telegram_chat_id` — BWS org `f0849278-925b-4f1c-b5dc-b4450099a630` → Keychain (Telegram only) → .env (DEV_MODE only). Keychain entry `neon-empire-bot`/`bitwarden-secrets-manager`.
- **Key files:** `backend/core/scheduler.py` · `backend/core/weather_signals.py` (gates/contrarian/funnel — audit before touching) · `backend/data/kalshi_client.py` · `backend/core/settlement.py` · `backend/secrets_broker.py` · `backend/api/settings_router.py` (hot-reload city modes) · `settings.json` · `recovery_queue.json` · `health_check_daily.py`.
- **Ops:** `healthcheck` / `botstart` / `botstop` (launchctl-based, ~/.zshrc). Liveness: `lsof -ti:8000` (verify with `ps aux` — Brave port collision). Manual recovery: `kill $(lsof -ti:8000); nohup .venv/bin/python run.py >> logs/trading_bot.log 2>&1 &` (RECOVERY.md).
- **Logs rotate/multiply confusingly** — trading_bot.log, app.log, agent002.log, agent002.error.log have each been "current" at some point; always `ls -la logs/*.log` and trust mtime before tailing.
- **Improvement prompts:** `weather-bot-improvement-prompts.md` (6 prompts; several now effectively executed via the Jul 5–10 engagement — reconcile ≈ done, process mgmt ≈ done).

## Next Steps

1. **Confirm the 10-row orphan backfill completed** and get the final tied-out realized P&L (−$5.85 pending).
2. **Investigate NYC's real settlement station** (KNYC isn't a real ICAO code) before trusting/fixing NYC coordinates.
3. **Let the bot run untouched ~1 week (≈Jul 17)** — first clean-bot run ever: coordinate bug, ghost trades, win/loss flips, and outage causes all fixed.
4. **After that week: re-examine LA's win rate** against correct LAX coordinates — testable hypothesis: was LA's poor performance the location bug, or no edge?
5. **KXLOW market audit** — same infra, ~2× daily opportunities; deferred until the coordinate fix has a track record.
6. Confirm `health_check_daily.py`'s hardened P&L-sanity timeout handling is fully deployed.
7. Execute the **Isobar rename** in code/UI (still "Neon Empire / WX Console v5") — cosmetic, low priority.
8. auto_tune stays OFF — 50+ trades per parameter + human approval required before enabling.
9. Verify `cancel_order` V2 migration happened (approved Jul 3; not mentioned since).
10. Open forensic question: the 00:00:01-vs-01:00 shutdown — revisit only if an unexplained shutdown recurs.
11. Backlog: contrarian_watch evaluation (needs >55% on 30+ signals); always-on device (Mac Mini/VPS) as bankroll grows.
