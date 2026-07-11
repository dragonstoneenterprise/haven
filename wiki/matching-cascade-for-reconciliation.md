---
created: 2026-07-11
updated: 2026-07-11
stage: wiki
source: "[[floatless-reconciliation-and-cashback-system]]"
---

# Matching cascade: reconciling two systems without a shared key

When two data sources describe the same real-world events but don't share a primary key (a broker's orders vs. a bank's charges, an ad platform vs. a CRM), don't try one fuzzy match — use a priority-ordered cascade of matchers, strongest signal first.

Floatless's version, in order:
1. Exact ID match (when one exists)
2. A secondary unique identifier from the other system (tracking number)
3. Amount + date-window match
4. A descriptive field match (item/model number)
5. Grouped/aggregate match (sum of several records vs. one bundled charge)

Each method only runs on records the earlier methods didn't claim. This took match rate from 77% to 97% — most of the gain came from adding the *second* method, not from making the fuzzy amount-matching smarter.

## The rule
Order your matchers from most-certain to least-certain, and let each one only see what's left. Don't try to build one clever fuzzy-match function — build several dumb, confident ones and chain them.
