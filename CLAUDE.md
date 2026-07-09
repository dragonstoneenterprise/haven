# SecondBrain Vault

Karpathy-style knowledge pipeline in plain markdown (Obsidian-compatible, no plugins). Knowledge flows through **stages**, never topic folders.

## Pipeline

```
inbox/  →  projects/  →  output/  →  wiki/
capture     active WIP    shipped     evergreen
```

## Conventions

### Stage flow is one-directional
`inbox → projects → output → wiki`. Nothing skips stages. A raw capture never goes straight to wiki; a project note never jumps past output.

### RULE: wiki is harvest-only
Wiki articles are **ONLY** harvested from finished items in `output/` — never written directly or speculatively. If it hasn't shipped, it hasn't earned a wiki entry.

### projects/ files are running logs
One file per project, always with these three sections:

```markdown
## Current State
## Decisions
## Next Steps
```

They serve as persistent handoff context — anyone (human or agent) should be able to pick up a project cold from its file.

### Filenames
- kebab-case, always
- **no dates in filenames** — dates live in YAML frontmatter:

```yaml
---
created: 2026-07-09
updated: 2026-07-09
stage: projects
---
```

### _index.md maps
Every stage folder has an `_index.md` — an auto-maintained map of its contents. **Update it whenever files are added, moved, or removed.**

### Inbox hygiene
- inbox/ is zero-friction: no organizing, no naming rules on capture
- Items older than **2 weeks** get triaged: promote into a project or delete. No limbo.

## Slash commands

| Command | Purpose |
|---|---|
| `/harvest` | Extract lessons from un-harvested `output/` items into `wiki/` |
| `/triage` | Process `inbox/` — merge into projects, flag orphans, delete junk |
| `/status` | Compact cross-project summary from `projects/` files |
