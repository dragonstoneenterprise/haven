---
description: Compact cross-project summary of current state and next steps
---

Give me a cross-project status summary.

1. Read every project file in `projects/` (except `_index.md`).
2. For each project, pull the essentials from `## Current State` and `## Next Steps` — compress to 1-2 lines each, keeping concrete specifics (numbers, blockers, dates) over vague summaries.
3. Output one compact block per project:

```
### <project-name>  (updated: <frontmatter date>)
Now:  <current state, 1-2 lines>
Next: <next steps, 1-2 lines>
```

4. Flag any project with empty sections or an `updated` date more than 2 weeks old as **stale**.
5. Read-only: do not modify any files.
