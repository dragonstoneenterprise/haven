---
created: 2026-07-09
updated: 2026-07-11
stage: projects
---

# gold-bot — HERMES (macro-regime gold system) + wealth allocation

HERMES is a macro-regime gold trading system: reads geopolitical WAR/PEACE signals from a macro collector, tags regimes, trades gold directionally (WAR_ESCALATING = SHORT, PEACE_EMERGING = LONG). Owned by the operator (Egypt-based; EGP collapsing 20–30%/yr — needs USD income + a house down payment, ~$10–15k in 5 years).

This file also carries the adjacent wealth-allocation decisions (Roth IRA, account structure, capital plan) since they're driven by the same goal HERMES serves — and the deferred future multi-asset bot **ATLAS**.

Scripts: `~/.hermes/scripts/` (macro_collector.py, xauusd_fetcher.py, regime_tagger.py, backtest_hermes.py, hermes_daily_signal.py). Signals via StrategyFactory API (`http://127.0.0.1:8765/api/strategies`). Telegram supergroup `-1004411966791`, Topic 90. Deployment target: **Bitget** XAU/USDT perpetuals, personal account (not yet opened).

Naming note (from isobar chat, Jul 10): rename to **"Assay"** was chosen but not executed — code/branding still says HERMES.

## Current State

*(as of this chat — most recent/only archive)*

- **HERMES daily system: VALIDATED.** 9 trades/3yr, 77% win rate, 4.75 profit factor, −2% max drawdown. Live at **$47.49**, manual execution — too small to run the bot for real yet.
- **HERMES 4H variant: BUILT AND REJECTED this session.** 3,104 4H bars (2024–2026; yfinance caps 1H data at 2yr so this isn't a full 3yr sample), 61 trades, 5.1% return, PF 1.13, WR 41%, DD 6.9%. Failed the abort threshold (needed PF ≥1.8, ≥12–15% return). Root cause: PEACE longs averaged R=−0.12 (a drag); WAR shorts averaged R=+0.33 and only worked because gold rose $2,308→$5,578 through the period — i.e., the edge may be a bull-market artifact, not a real regime signal.
- Regime distribution (4H): 87% UNCERTAIN, 6.4% PEACE, 6.4% WAR.
- `hermes_signals.json` is live. Cron for the daily `--send` step is written but **not added**.
- **Bitget account: NOT OPENED.** Planned $500 deposit doesn't exist yet — realistic path is save $200–300/mo, deploy $300+ by month 3–4.
- **Roth IRA 1 (Fidelity):** GLD $600 bought this session. $513.26 cash remains — plan is VTI +$380, IBIT +$130, keep $3.26. **XOVR sale status unclear** — flagged as "sell" repeatedly but cash math suggests it may already be sold. Needs verification.
- **Roth IRA 2 (iTrustCapital):** KMG $191 held. Optional +$100 if the custodian allows.
- Weather bot (isobar) holds $250 on Kalshi, personal account (Kalshi forbids business accounts) — do not raid this for Bitget funding.
- **Total capital ≈ $2,540** across everything.
- **Blocked:** Bitget deployment blocked by capital shortage (not a technical blocker). Roth buys blocked only by the operator taking action.

## Decisions

### Roth allocation — arrived at through iteration
Settled on **hybrid 45% gold / 55% growth**: GLD $600 + VTI ~$750 + IBIT ~$201 + KMG $191–291. History: first recommendation was 62% gold-heavy (EGP hedge) → operator pushed back, "gold is wealth preserve not growth, I need income" → flipped to 32% gold → operator pushed back again, "what if war becomes more likely, I want to profit both ways" → settled on hybrid. Reasoning: gold hedges EGP collapse/oil-spike/war; VTI+IBIT compound during peace; **HERMES itself profits in both directions**, so the Roth doesn't have to carry that job alone.
*Lesson: the operator's stated goal (income) overrides thematic conviction (gold supercycle) — don't let a strong macro thesis override what the person actually said they need.*

### Daily system kept over 4H
4H failed its own kill criteria (PF 1.13 < 1.8 target, 5.1% < 12–15% target return). Considered and rejected: a WAR-only variant (would be curve-fit to the 2024–26 bull run, untested regime dependency) and inverting the PEACE signal (same overfit risk). *Same discipline that killed an earlier BTC swing system (50% WR, 1.216 PF, ~$4.51/mo on $500) — built, honestly backtested, abandoned rather than deployed. Saved $300 by not shipping a marginal system.*

### Platform: Bitget
0% maker / 0.0065% taker promo (**unverified as of this chat — confirm still active before committing**), saves ~$200–300/yr vs Binance, XAU/USDT perps, works in Egypt, $50 minimum. Rejected: XAUT on Base (spot only, can't short — breaks the strategy), iTrustCapital (custodian only, no API), Solana trade.xyz (spread/automation uncertainty), OANDA (backup, slower). An earlier session had recommended Binance before Bitget research superseded it — flag that reversal if it resurfaces.

### Account structure
- **All trading stays personal** — Bitget under the operator's own name, matching Kalshi's forced-personal structure.
- **Dragonstone Enterprises LLC = infrastructure provider only** — hosts bot servers; operator invoices himself $75–100/mo hosting fee for a clean audit trail + deduction. Not yet set up.
- **Floatless is 100% LLC** (business ops tool) — different from the trading bots, which are personal.
- **Roth is personal by law** — funded via LLC distributions → personal → Roth contributions.

### No individual AI stocks
Thesis (Jordi Visser): bullish market overall but short hyperscalers (CapEx air-pocket risk, open-source catching up — GLM 5.2 ≈ Opus 4.8 at 1/6 cost), long semis/infrastructure. VTI already owns both sides of that trade; picking individual names at ~$2.5k total capital has no edge. **The operator's edge is HERMES, not stock selection.**

### ATLAS — deferred, gated
Future multi-asset bot (Cowork-style, inspired by Andrew Antiles/Graystone). **Explicitly gated behind:** HERMES live 3+ months matching its backtest within ±10%, AND capital ≥$3,000. Full 6-phase plan filed but not started.

Architecture skeleton: `~/atlas/` — `core/` (risk_manager, position_sizer, correlation_filter, execution via ccxt), `strategies/` (trend_follow_4h for gold/oil, momentum_1h for BTC, mean_reversion_15m for indices), `reporting/` (morning_brief, evening_report), `config.yaml`. Signal schema: `{asset, direction, confidence, stop_pct, target_pct, timeframe}`. Risk rules: hard 1% stop, max 2% portfolio risk, ATR-based sizing, correlation >0.7 blocks stacking positions, −3% daily halt, max 3 concurrent positions. **The Claude layer reports and flags only — never trades or changes params directly.**

The 33/33/33 land/business/liquid allocation framework is also deferred until capital ≥$3,500; Egyptian farmland was rejected outright (EGP/political/legal risk).

### Corrections & lessons
- 4H fetcher hit a mixed-timezone CSV bug — fixed by parsing with `utc=True` then stripping tz.
- yfinance 1H data caps at 2 years, so the 4H backtest covers 2024–2026 rather than the planned full 3yr — sample is still valid, just shorter than intended.
- Post-GLD-buy "gold is dropping" panic — correct response was to hold; daily noise is irrelevant to a 5–20yr thesis.
- (Referenced from weather-bot memory, not this project) phantom P&L +$576.88 corrected to ≈−$5.85 — same lesson applies here: verify P&L against settled trades, never a displayed total. See [[isobar]] and the wiki article on source-of-truth accounting.

### Reference
**Backtest benchmarks (daily):** 77% WR / 4.75 PF / −2% DD / 9 trades per 3yr. **Kill criteria:** PF < 1.5 → don't deploy; live WR deviates >15% from backtest after 20 trades → pause.

**Bitget deployment params:** XAU/USDT, 1x leverage only, market entry, 1.5% stop, 4% target, cron daily 8:00 UTC, signal sourced from StrategyFactory API, trade log to `~/hermes/trades_hermes_gold.json`, alerts to Telegram Topic 90.

**Credentials:** Bitget API keys in `~/.bitget/credentials.env` (chmod 600); env vars `BITGET_API_KEY` / `BITGET_API_SECRET` / `BITGET_PASSPHRASE`; trading-only permissions, **never withdrawal**. Bot status: `lsof -ti:8000`. Shell controls: `trade` / `botstop` / `botstart` / `botstatus`.

**Bitget bonuses:** $200 welcome = BTC-futures-only coupon (skip, doesn't fit gold strategy). $6,000 deposit&trade ladder = real USDT ($1k net deposit → $50; $100 spot volume → $400; $300k futures volume → $6,000 — reachable around month 10–12 of bot operation).

**Telegram:** supergroup `-1004411966791` — Topic 75 (forward testing), 83 (Quant Lab HQ), 90 (HERMES/crypto).

**Cross-project context:** Dragonstone Enterprises (WY) + Cryptic Dragon (CA) LLCs; Floatless repo `dragonstoneenterprise/floatless`, Supabase ref `plzneojxlqleaeprbygk` — see [[floatless]].

## Next Steps

1. **Verify XOVR sold**, then execute Fidelity buys: VTI +$380, IBIT +$130. (Operator action, ~10 min.)
2. Open the Bitget **personal** account: email + 2FA + KYC (no VPN, real info), create an API key labeled `HERMES_BOT`, trading-only permissions.
3. Capital plan: save $200–300/mo, deposit $300+ to Bitget (USDT via Tron) by month 3–4. **Do not raid the weather bot's $250.**
4. Verify the Bitget 0% maker fee promo is still active before committing capital.
5. Add the cron job for `hermes_daily_signal.py --send` (written, not added).
6. Build the Bitget execution bot: daily 8:00 UTC, reads StrategyFactory, atomic entry+stop+target, **testnet paper trade for 1 week before going live.**
7. Set up the LLC hosting invoice ($75–100/mo, personal → LLC).
8. Open questions: can iTrustCapital take +$100 into KMG? Does Bitget offer index perps (matters for ATLAS phase 3)? Which oil symbol on Bitget (USOIL vs WTI)?
9. File the ATLAS 6-phase plan properly; revisit only after both HERMES gates pass (3mo live matching backtest ±10%, capital ≥$3,000).
