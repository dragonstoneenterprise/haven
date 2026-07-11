---
created: 2026-07-11
updated: 2026-07-11
stage: output
harvested: true
harvested_date: 2026-07-11
source_project: "[[floatless]]"
---

# Floatless: order reconciliation & cashback tracking

Shipped and live in production. Matches BFMR orders against credit card statement CSVs to surface real profit after cashback, across two LLCs and multiple cards.

## What shipped
- A 5-method priority-ordered matching cascade (Amazon order ID → tracking number → charge amount within a date window → model number → grouped same-day charges) that took reconciliation from 77% to 97%.
- Per-order cashback storage (not looked-up-from-card), so card reassignments never alter historical profit math.
- A "Profitable Credit" metric that only counts credit on cards with 5%+ cashback or an active welcome bonus — because thin BFMR margins mean lower-rate cards aren't worth counting as buying power.
- A near-miss: an orphaned duplicate-looking card record almost got merged into an active account, which would have silently doubled a key metric. Caught by checking account scope and order count before treating anything as a duplicate.

## Why it matters beyond this project
The matching cascade — try the strongest, most specific signal first, fall back to progressively fuzzier matches — is a reusable pattern for reconciling any two systems that don't share a primary key (a broker vs. a bank, an ad platform vs. a CRM, etc.). The duplicate-detection near-miss is also a reusable caution: two records that look alike aren't necessarily duplicates — check ownership scope and activity, not just field similarity.

Full detail: [[floatless]]
