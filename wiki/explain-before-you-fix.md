---
created: 2026-07-11
updated: 2026-07-11
stage: wiki
source: "[[isobar-v2-migration-and-reliability-overhaul]]", "[[floatless-reconciliation-and-cashback-system]]"
---

# Explain a wrong number before you fix it

When a number looks wrong, the fast move is to patch the display so it looks right. That's the move to resist.

In isobar, refusing to accept "the dashboard is just wrong, fix the display" and instead demanding an explanation is what surfaced a structurally decoupled accumulator that had drifted for weeks — and the same discipline, applied twice more, caught a $154 win/loss-flip bug and a 17°F coordinate error that had been silently corrupting an entire city's trading signals for 81 days.

In floatless, a duplicate-looking record was nearly merged into an active account — which would have silently doubled a key metric — until checking *why* it looked like a duplicate (account scope, order count) instead of just merging it.

## The rule
Before changing a number to match your expectation, find out why it doesn't match. "Fix the display" almost always means "hide the bug." The bug is usually more consequential than the number.
