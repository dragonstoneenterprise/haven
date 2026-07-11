---
created: 2026-07-09
updated: 2026-07-11
stage: projects
---

# isobar — Kalshi weather trading bot

Python/FastAPI bot that trades Kalshi daily high-temperature prediction markets across 9+ US cities, using a dual-model weather ensemble (GFS + ECMWF) to find mispricings. Runs locally on the MacBook Air — not a cloud service; dies if the Mac sleeps or crashes.

Stack: Python 3.11, FastAPI/Uvicorn, APScheduler, SQLite (`tradingbot.db`), Bitwarden Secrets Manager (credential broker), Telegram (alerts), launchd (scheduler).
Local path: `~/neon-empire/agents/agent-002-weather` · no public repo · dashboard at `http://127.0.0.1:8000/` when running.

## Current State

*(as of Jun 24, 2026 — earliest archived chat; newer info supersedes)*

**Financials:**
- Kalshi account: **$303.92 cash, 0 open positions** (source of truth).
- Bot DB: bankroll $303.92, P&L listed +$395.88 over 175 trades — **P&L unreliable** (inflated by historical ghost trades, never reconciled against Kalshi). Real verified profit: unknown; Kalshi history only visible back to Jun 15.

**Infrastructure:**
- launchd auto-start active — starts 12:00 UTC (7am ET / 2pm Cairo), stops 01:00 UTC (8pm ET).
- Bitwarden machine account connected via macOS Keychain (`neon-empire-bot` / `bitwarden-secrets-manager`).
- Health watchdog every 5 min with Telegram alerts. Dashboard live, wired to real data (`/api/risk`, `/api/stats`, `/api/trades`, `/api/learning/summary`).
- Bot stopped (scheduled), 0 pending trades, recovery queue empty, all ghost trades voided.

**Cities:**
- Live: Miami, LA, Denver, Austin
- Paper: Chicago, Dallas, Phoenix, Atlanta, Seattle
- In settings.json but unconfirmed: NYC, Minneapolis, Boston — NYC historically 0W/4L, should be paper but not yet changed.

**Open contradictions (unresolved):**
- DB says 175 trades / +$395.88; Kalshi history (Jun 15–24) shows far fewer. Gap unresolved.
- `learning_nightly` runs nightly but nobody has read the function — it could be doing anything.
- NYC is in the active city list despite being historically blacklisted.

## Decisions

### Signal gate — the core edge (parameters FROZEN; do not change without audit)
- Dual-model agreement: GFS vs ECMWF must diverge **<3.5°F** — blocks uncertain forecasts.
- Favorable-side model probability **≥0.65** (raw, unclamped) — audit showed pre-fix winners had model_prob 0.74–0.95 while post-fix losers had 0.13–0.29; the gate separates genuine model/market disagreement from noise.
- Max entry price **0.45** — expensive NO trades (>0.55) lost money even at 55% win rate due to inverted payout math; the >0.55 bucket cost −$21 historically.
- Min edge **8%**. Kelly fraction **25%**, ~$8 flat size.
- `learning_nightly` must not touch these — **unconfirmed**, highest-priority unknown.

### Strategy decisions
- **Unanimous ensemble filter: kept.** (Skip when model prob <0.10 OR >0.90 AND market agrees.) Initially suspected of blocking winners; audit showed it never touched the cheap-YES winning pattern — that setup (model 90%+, market 5–17%) simply stopped appearing in summer. Do not remove.
- **NYC cut from live** — historically worst city (0W/4L, −$15.90). Never re-add to live without clean paper evidence.
- **City expansion Phase 1** — Dallas, Phoenix, Atlanta, Seattle added in paper mode.
- **Contrarian watch** — trades where model_prob <0.10 but market price <0.20 are logged to signals table tagged `contrarian_watch`, paper only. Audit Finding B: +$180 hypothetical at 60% win rate over 4 days — too thin to trade yet; building dataset.
- **Not selling the bot** (decided Jun 24) — alpha decay: too many buyers running the same strategy on thin Kalshi weather markets would erode the edge.

### Engineering decisions
- **Bitwarden over .env** — secrets never in plaintext. Runtime: macOS Keychain bootstrap token → Bitwarden SDK. All four trading secrets live in Bitwarden Secrets Manager.
- **Dashboard exposes only safe knobs** (bankroll, sizing, Kelly, cities, alerts). Signal-gate parameters never appear in the UI or settings.json.
- **launchd over cron** — Mac-native, handles restarts, integrates with login session. Existing cron for `check_series.py` (noon daily) left untouched.
- **409/410 handling** — both caught *before* DB write; pre-flushed trade row deleted on rejection, no recovery-queue entry. Ghost trades were the #1 operational bug.
- **4pm ET cutoff** — bot skips same-day markets after 16:00 ET to prevent 410s on closed markets.

### Corrections & lessons
- **Ghost trade bug (major, recurring):** trade was written to DB before the Kalshi order; on 409/410 rejection the fake open trade blocked future real trades via position cap. Fixed pre-commit. *Lesson: verify against Kalshi Positions tab, not bot DB.*
- **"Unanimous filter was blocking winners" — wrong.** The filter only fires when model AND market agree (expensive side); the winning pattern died seasonally. *Lesson: don't change working filters on symptoms; audit first.*
- **"index.html was a separate product from the mockup" — wrong.** WeatherDesk Console v1.0.0 and the neon mockup were the same file; a session was wasted cleaning up an accidentally revived `dashboard.html`. *Lesson: check file contents before assuming architecture.*
- **"Build script regenerates the .tar.gz" — wrong, twice.** It updated `dist/` but never re-tarred; the archive stayed stale. Caught by an independent `tar -xzOf` grep. *Lesson: verify the actual artifact, not the staging folder.*
- **DB P&L +$395.88 likely inflated** by the ghost-trade era. *Lesson: DB is not source of truth; the Kalshi account is.*
- **Broker=FAIL on startup despite working secrets:** `_bw_client` uses `@lru_cache(maxsize=1)` and serves a cached failure forever; `cache_clear()` never called before the health check. *Lesson: lru_cache on auth clients needs an explicit invalidation path.*
- **Bitwarden 404s on a working machine account:** the new account had access to the "Bot Agents" project, not "Neon Empire agents" where the secrets live. *Lesson: machine accounts need explicit project grants, not just org membership.*

### Reference
**Signal gate env names:** `WEATHER_MIN_FAVORABLE_PROB=0.65` · `WEATHER_MAX_ENTRY_PRICE=0.45` · `WEATHER_MIN_EDGE_THRESHOLD=0.08` · `WEATHER_GFS_ECMWF_DIVERGENCE_THRESHOLD=3.5` (°F) · `KELLY_FRACTION=0.25`

**Bitwarden secret names:** `kalshi_api_key`, `kalshi_api_secret`, `telegram_token`, `telegram_chat_id`. Keychain: account `neon-empire-bot`, service `bitwarden-secrets-manager`. Org ID: `f0849278-925b-4f1c-b5dc-b4450099a630`.

**Key files:**
- `backend/core/weather_signals.py` — signal generation, gate logic (**do not touch without audit**)
- `backend/core/scheduler.py` — job scheduling, trade execution, health check, 409/410 handler
- `backend/core/settlement.py` — settlement with 6hr observation fallback
- `backend/secrets_broker.py` — credential loading; priority BWS → Keychain → .env (DEV_MODE only)
- `backend/static/index.html` — dashboard frontend · `backend/api/connections_router.py` — Bitwarden/Kalshi/Telegram status
- `settings.json` — runtime config (cities, paper mode, safe knobs only)
- `recovery_queue.json` — retry queue; clear with `echo "[]" > recovery_queue.json` when stale
- `tradingbot.db` — trade history (not source of truth for balance)
- `~/Library/LaunchAgents/com.neonempire.weatherbot.plist` (start 12:00 UTC) / `...weatherbot.stop.plist` (stop 01:00 UTC)

**Shell commands:** `trade` (manual start) · `botstop` (kill + unload launchd) · `botstart` (load + start) · `botstatus` (check state)

**Trade patterns from audit:**
- Winning: model_prob ≥0.70 favorable side, entry <0.35, genuine model/market disagreement — what the 0.65 gate selects for.
- Losing: model_prob 0.13–0.29, entry <0.20 — 8% edge satisfied on noise, loses 75–85% of the time — what the gate blocks.

## Next Steps

1. **Confirm `learning_nightly` scope** — read the function; verify it cannot modify signal-gate parameters. Highest-priority unknown. (Claude Code prompt: "Show me every file and parameter that learning_nightly reads and writes. Does it touch any value in settings.json or config.py related to signal gates?")
2. **Set NYC to paper** — add `nyc` to `WEATHER_PAPER_CITIES` in settings.json (one line; it's currently in the active scan list).
3. **Fix startup health check** — call `_bw_client.cache_clear()` before the broker test in `run_startup_health_check`.
4. **Reconcile DB P&L against Kalshi** — settlement history API vs `tradingbot.db`. Do this before any sizing decisions.
5. **Verify first clean end-to-end execution** — `botstatus` at 2pm Cairo, watch for `[TRADE]` with no 409/410, confirm real fill in Kalshi Positions.
6. **Evaluate contrarian_watch** after 3–4 weeks of paper data — if win rate >55% on 30+ signals, consider a separate small-size live strategy.
7. **Minneapolis & Boston** — in the city list but never evaluated: add lat/lon and paper-mode them, or remove.
8. **Consider an always-on device** (Mac Mini / VPS) — Mac sleep kills the bot. Not urgent; revisit as bankroll grows.
