---
created: 2026-07-09
updated: 2026-07-11
stage: projects
---

# isobar — Kalshi weather trading bot

Python trading bot ("Isobar", agent-002-weather) that autonomously trades Kalshi daily high-temperature markets across US cities. Signals from a dual-model ensemble (GFS 31-member + ECMWF 51-member via open-meteo): trade only when both models agree within 3.5°F, then buy the favored side when edge ≥ 8%. Runs locally on the MacBook Air — not a cloud service; dies if the Mac sleeps or crashes.

Stack: Python 3.11, FastAPI/uvicorn (dashboard http://127.0.0.1:8000), SQLite (`tradingbot.db`), APScheduler, Bitwarden Secrets Manager (via macOS Keychain `neon-empire-bot`/`bitwarden-secrets-manager`), Telegram alerts, launchd.
Local path: `~/neon-empire/agents/agent-002-weather` · no public repo · Kalshi account mushudragon01, Basic API tier.

## Current State

*(as of Jul 3, 2026 chat — newest archived)*

- **Bot RUNNING**, started manually via `nohup .venv/bin/python run.py` (check with `lsof -ti:8000`, NOT pgrep — see Corrections).
- **launchd autostart BROKEN/unloaded** (`Load failed: 5: Input/output error`) — bot will NOT survive a reboot. Fix written (Prompt 2), not run.
- **Bankroll:** $293.59 cash + $9.51 in 4 open positions (was $303.92 on Jun 24).
- **Live win rate: 24W/8L = 75% (+$104.71)** — but n=32, true rate 60–90%. Paper: 37W/7L (inflated, ideal fills). Legacy NULL-mode rows: 48W/78L = 38%, contaminated with ghost trades — they poison all aggregate stats.
- **DB claims +$398 P&L; reality ≈ −$6 to +$10 vs deposits. NOT reconciled.**
- **V2 order endpoint migration DONE and verified** with real fills Jul 2–3, including partial fills (LAX: requested 7, filled 4 and 3) and one live zero-fill auto-void.
- Cities live: miami, los_angeles, denver, atlanta, chicago, dallas, phoenix, seattle (+austin?). Paper: new_york, minneapolis, boston, austin. **Contradiction: settings.json shows austin=paper but the Jun 24 handoff said austin live — settings.json is authoritative.**
- 6 detailed Claude Code improvement prompts written (`weather-bot-improvement-prompts.md`) — **none executed yet**.

**Open questions:** exact initial deposit for reconciliation baseline (~$300, unconfirmed); austin live-vs-paper; whether `GET /portfolio/orders` needs migration (changelog says legacy costs may rise).

## Decisions

### Signal gate — the core edge (FROZEN; do not change without audit)
- GFS vs ECMWF divergence **≤3.5°F**; min edge **8%**; favorable-side model prob **≥0.65** (raw, unclamped); entry price **∈ [0.03, 0.45]**; buffer **≥1.2°F** from threshold; yes_price ∈ [0.05, 0.95]; 4pm-ET same-day cutoff; MAX_TRADES_PER_SCAN=5; MAX_TOTAL_PENDING=5; $500 weather allocation cap.
- Sizing: Kelly 25% clamped by max_trade_size 15 → effectively flat ~$8/trade.
- Why 0.65: audit showed winners had model_prob 0.74–0.95, losers 0.13–0.29 — the gate separates genuine model/market disagreement from noise. Why max 0.45 entry: expensive NO trades (>0.55) lost money even at 55% win rate (that bucket cost −$21).
- **`learning_nightly` confirmed SAFE** (Jul 3) — seeds promotions only, owner must approve via API; cannot drift signal gates. *(Was the #1 unknown from Jun 24 — resolved.)*
- **DO NOT tune strategy now** — 75% live at ~$0.36 avg entry is strong; changing gates resets a small sample. Win rate alone is the wrong dial (buying at $0.90 wins 90% and loses money). Let it run 2–3 weeks / ~100 live trades before any sweep.

### Strategy decisions
- **Unanimous ensemble filter: kept.** (Skip when model prob <0.10 OR >0.90 AND market agrees.) Audit showed it never blocked the cheap-YES winners — that setup (model 90%+, market 5–17%) died seasonally in summer. Do not remove.
- **NYC cut from live** — worst city historically (0W/4L, −$15.90); never re-add without clean paper evidence. Now in paper.
- **Contrarian watch** — model_prob <0.10 + market <0.20 logged paper-only as `contrarian_watch` (+$180 hypothetical @60% over 4 days — too thin). Evaluate after 3–4 weeks; needs >55% on 30+ signals to consider small-size live.
- **Not selling the bot** (Jun 24) — alpha decay: buyers running the same strategy on thin Kalshi weather markets erodes the edge.

### Kalshi V2 migration (Jul 2–3) — root cause of 2+ weeks of zero fills
- **`place_order` → V2 `POST /portfolio/events/orders`** — legacy `/portfolio/orders` returns 410 Gone (deprecated May 2026, killed ~late June).
- **V2 side/price mapping is YES-relative:** `yes` → `bid` @ price/100; `no` → `ask` @ (100−yes_price)/100. Scheduler passes the bought side's price in cents, so NO uses the complement. **Getting this wrong = instant bad fills.**
- **`time_in_force: immediate_or_cancel`, NOT good_till_canceled** — no fill-polling or resting-order cleanup exists, and cancel_order is still on a deprecated path; GTC could rest unfilled → inverted ghost (Kalshi position the DB doesn't know). IOC matches old market-order semantics. GTC rejected deliberately.
- **Fill verification in scheduler:** fill_count==0 → void DB row (mirrors 410 path); partial → trade.size=fill_count; missing fill_count → KEEP row + ERROR log — never auto-void on ambiguous data (deleting on parse failure risks voiding real positions).
- `cancel_order` migration to `DELETE /portfolio/events/orders/{id}` approved but **not yet done** (3-line change; `kalshi_client.py:101` still on deprecated path).

### Engineering decisions
- **Bitwarden over .env** — secrets never in plaintext; Keychain bootstrap token → Bitwarden SDK at runtime.
- **Dashboard exposes only safe knobs** (bankroll, sizing, Kelly, cities, alerts); signal-gate parameters never in UI or settings.json. City modes hot-reload via `settings_router.py`.
- **launchd over cron** — Mac-native, handles restarts; existing cron for `check_series.py` (noon daily) untouched.
- **409/410 caught before DB write**; pre-flushed row deleted on rejection. 4pm ET cutoff prevents 410s on closed same-day markets.
- **Contrarian-watch filter fix (Jul):** only block when intended direction is YES (`calculate_edge` returns "up"). It had been blocking ALL legitimate NO trades where model_yes<0.10 and yes_price<0.20 — silently killed ~3 days of trading (62 Miami signals @ 0.828 edge, 0 executed).
- **Health-check no-trades alarm** filters to `Signal.executed==True` and `Trade.mode!='paper'` so voided signals don't false-alarm.
- **Self-healing architecture decided, not built** — 4 layers: **Sentinel** (deterministic: fill-rate invariant, error-monotony detector, Kalshi changelog watcher, daily demo-env canary order), **Medic** (AI diagnosis on alarm, read-only), **Surgeon** (AI patches PROPOSED, human /approve via Telegram, file allowlist, test+canary gate), **Warden** (24h post-patch watch + auto-rollback). Fully-autonomous self-patching rejected: a wrong bid/ask fix places bad trades silently with real money.
- **Rejected:** parallel dispatch of improvement prompts (3→4→5 are sequential); Cowork for heavy work (5–20× token burn vs chat); switching models as a fix ("model choice isn't the bottleneck").

### Corrections & lessons
- **Ghost trade bug (major, recurring):** trade written to DB before Kalshi order; on 409/410 the fake open trade blocked real trades via position cap. Fixed pre-commit. *Verify against Kalshi, not bot DB.*
- **"410s = market closed" — wrong:** it was endpoint deprecation. 100% failure across all cities including next-day markets was the tell. *Uniform failure signature = systemic, not market conditions.*
- **"Unanimous filter was blocking winners" — wrong:** the winning pattern died seasonally. *Don't change working filters on symptoms; audit first.*
- **`pgrep -f "python run.py"` lies** — macOS Homebrew binary is capital-P `Python`; hours lost fighting our own live process over port 8000. *Use `lsof -ti:8000`.*
- **"No fills since June 19" — wrong:** Kalshi app History was showing old entries; DB showed 98W/91L. *Verify from DB, not app scroll position.*
- **Low win-rate impression** came from aggregating legacy ghost rows with live rows. *Segment by mode before judging performance.*
- **broker=FAIL at startup:** current theory (Jul 3, unverified) — `_check_broker` imports a nonexistent `get_secret` from secrets_broker (real accessor `broker._value()`); ImportError swallowed → permanent false FAIL. Check: `.venv/bin/python -c "from backend.secrets_broker import get_secret"`. *(Supersedes the Jun 24 lru_cache theory — `_bw_client` `@lru_cache` caching a failed client is still worth fixing regardless.)*
- **`botstatus` right after `botstart` reads STOPPED** (launchd registration lag).
- **"index.html was a separate product" — wrong** (same file as the mockup); **"build script re-tars the archive" — wrong twice** (only updated `dist/`; caught by independent `tar -xzOf` grep). *Verify the actual artifact; check file contents before assuming architecture.*
- **DB P&L (+$398) unreliable** — ghost-trade era inflation. *Kalshi account is source of truth.*
- **Bitwarden machine-account 404s** — account had access to "Bot Agents" project, not "Neon Empire agents" where secrets live. *Machine accounts need explicit project grants.*

### Reference
- **Key files:** `backend/core/scheduler.py` (weather_scan_and_trade_job, health checks, 409/410/fill handling — ~250-line job) · `backend/core/weather_signals.py` (gates, contrarian, funnel — do not touch without audit) · `backend/data/kalshi_client.py` (place_order V2; cancel_order at :101 still deprecated) · `backend/core/settlement.py` (6hr observation fallback) · `backend/secrets_broker.py` (priority BWS → Keychain → .env DEV_MODE only) · `backend/api/settings_router.py` (hot-reload city modes) · `backend/api/connections_router.py` (status endpoints) · `backend/static/index.html` (dashboard) · `settings.json` (cities, paper mode, safe knobs only) · `recovery_queue.json` (clear with `echo "[]" > recovery_queue.json`) · `tradingbot.db`.
- **Env names:** `WEATHER_MIN_FAVORABLE_PROB=0.65` · `WEATHER_MAX_ENTRY_PRICE=0.45` · `WEATHER_MIN_EDGE_THRESHOLD=0.08` · `WEATHER_GFS_ECMWF_DIVERGENCE_THRESHOLD=3.5` · `KELLY_FRACTION=0.25`.
- **Secrets (4):** `kalshi_api_key`, `kalshi_api_secret`, `telegram_token`, `telegram_chat_id` — BWS org `f0849278-925b-4f1c-b5dc-b4450099a630`; Keychain `neon-empire-bot`/`bitwarden-secrets-manager`.
- **Kalshi:** API amounts are CENTS. Ticker format `KXHIGH<CITY>-<YY><MON><DD>-{B<x>.5 = between band | T<x> = threshold}`.
- **Logs:** `logs/app.log` (main, daily-rotated), weather.log, trades.log, settlement.log, recovery.log. `trading_bot.log` DOES NOT EXIST.
- **Ops:** status = `lsof -ti:8000 && echo RUNNING || echo STOPPED`. Commands: `trade` / `botstart` / `botstop` / `botstatus` (zsh; botstatus unreliable). launchd plists: `~/Library/LaunchAgents/com.neonempire.weatherbot.plist` (start 12:00 UTC) + `.stop.plist` (stop 01:00 UTC).
- **Trade patterns (audit):** winning = model_prob ≥0.70 favorable, entry <0.35, genuine disagreement. Losing = model_prob 0.13–0.29, entry <0.20 — noise passing the 8% edge check, loses 75–85%.
- **Improvement prompts:** `weather-bot-improvement-prompts.md` — 6 prompts (1 reconcile, 2 process mgmt, 3 funnel+tests, 4 refactor, 5 observability, 6 strategy). Run in order, fresh session each, review Phase 0 before edits.

## Next Steps

1. Verify Jul 3 settlements — Telegram alerts + Kalshi cash/positions vs DB; first real P&L cycle through the V2 code.
2. Migrate `cancel_order` to `DELETE /portfolio/events/orders/{id}` (3-line, approved, not done).
3. **Run Prompt 1 (reconciliation)** — establish true P&L, void ghosts. Blocks all performance analysis. Need the exact initial deposit (~$300, unconfirmed) as baseline.
4. **Run Prompt 2 (launchd repair + lockfile + crash handler + botstatus fix)** — bot currently won't survive a reboot.
5. Fix broker=FAIL — verify the `get_secret` ImportError theory (Prompt 5 has detail).
6. Prompts 3→4→5 sequentially; Prompt 6 (strategy) only after reconcile.
7. Build Sentinel layer (especially changelog watcher + demo canary — would have caught the V2 outage in May), then Medic/Surgeon/Warden phases.
8. After 2–3 weeks / ~100 live trades: win-rate analysis sweep before any gate tuning (75% now, n too small).
9. Resolve austin live-vs-paper contradiction (settings.json is authoritative); check whether `GET /portfolio/orders` needs V2 migration.
10. Backlog: evaluate contrarian_watch dataset (needs >55% on 30+ signals); consider always-on device (Mac Mini/VPS) as bankroll grows.
