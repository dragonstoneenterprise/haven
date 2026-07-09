---
description: Process inbox/ — merge items into projects, flag orphans, delete junk
---

Triage the inbox.

1. Read every file in `inbox/` (except `_index.md`) and every project file in `projects/`.
2. For each inbox item, decide:
   - **Match**: it clearly relates to an existing project → merge its content into the right section of that project file (`## Current State`, `## Decisions`, or `## Next Steps`), update the project's `updated` frontmatter date, then delete the inbox file.
   - **Orphan**: real content but fits no existing project → do NOT delete. Flag it in your report and suggest either a new project file or a target; only create a new project if I confirm.
   - **Junk**: obviously stale, duplicate, or worthless → list it as proposed-delete and delete only what I confirm as junk.
3. Note the age of remaining items — anything older than 2 weeks (per `created` frontmatter or file mtime) must be resolved this pass: promote or propose deletion. No limbo.
4. Update `inbox/_index.md` and `projects/_index.md` to reflect moves.
5. Report: merged items (and where), orphans needing a decision, deletions proposed/made.
