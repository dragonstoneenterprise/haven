---
description: Harvest reusable lessons from shipped output/ items into wiki/ articles
---

Harvest the output/ folder into the wiki.

1. Scan every file in `output/` (except `_index.md`). Find items **not yet harvested**: frontmatter missing `harvested: true`.
2. For each un-harvested item:
   - Read it and extract the reusable, evergreen lessons — techniques, gotchas, patterns, decisions that would apply beyond this one artifact. Skip items with nothing generalizable; still mark them harvested with `harvested: true` so they aren't rescanned.
   - Write (or extend an existing) `wiki/` article per distinct lesson/topic. Kebab-case filename, no dates. YAML frontmatter with `created`, `updated`, `stage: wiki`, and `source: "[[<output-file>]]"`.
   - In the article body, link back to the source item in `output/`.
   - Set `harvested: true` (and `harvested_date: YYYY-MM-DD`) in the output item's frontmatter.
3. Update `wiki/_index.md` with any new articles and bump `output/_index.md` if harvest state is noted there.
4. Report: items harvested, wiki articles created/updated, items skipped as non-generalizable.

RULE: wiki articles come ONLY from `output/` items. Do not write wiki content from memory, speculation, or in-progress project work.
