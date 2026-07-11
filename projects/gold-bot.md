---
created: 2026-07-09
updated: 2026-07-11
stage: projects
---

# gold-bot — HERMES (rename target: Assay)

Gold trading bot, currently named HERMES, running against **Bitget** (vs isobar's Kalshi). Known so far only from isobar-chat context — no dedicated archive dumped yet.

## Current State

- **Completely parked** — not touched during the Jul 5–10 isobar engagement (as of Jul 10, 2026).

## Decisions

- **Rename to "Assay"** (gold purity testing) — decided, not executed. Rejected: Aurum, Midas (both scam-adjacent in existing trading-bot products), Solidus, Croesus (collide with established finance companies). The unified "Wyrm-" prefix family (Wyrmhoard/Wyrmwind) was rejected as "too cheesy." No umbrella/company name finalized.
- **Kept architecturally separate from isobar** — different platform (Bitget vs Kalshi), different asset class, different failure domain; one bot's bugs must not affect the other. Brand-level unification only.

## Next Steps

- Dump old HERMES/gold-bot chats into this file (use the handoff prompt).
- Execute the Assay rename when the project wakes up.
