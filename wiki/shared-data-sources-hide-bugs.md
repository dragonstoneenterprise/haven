---
created: 2026-07-11
updated: 2026-07-11
stage: wiki
source: "[[isobar-v2-migration-and-reliability-overhaul]]"
---

# A "narrow" bug fix can be feeding a bigger one

Isobar's settlement code was using the wrong weather station coordinates for a handful of cities — a scoped, low-stakes-looking bug in a fallback path. But that same coordinate table also fed live signal generation, the code that decides whether to place a real trade. Fixing "just the settlement fallback" would have left every live trading decision for that city silently wrong.

## The rule
Before scoping a fix to "just this one code path," check what else reads the same data source, config value, or shared function. A bug in a low-stakes path is often a symptom of a shared dependency that's also feeding a high-stakes one.

## Practical check
When you find a wrong value, grep for every caller of the function/table/config that produced it — don't assume the path you found it in is the only path that uses it.
