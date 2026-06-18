---
description: Audit & repair CLAUDE.md files against the repo's real conventions
argument-hint: "<empty for repo-local files, or 'global' to include ~/.claude>"
---

# /claude-md

Wraps `claude-md-doctor`. Cross-checks CLAUDE.md guidance against `patterns.json` / `audit.json` and proposes diff-gated fixes. Repo-local files only unless you pass `global`.

```
/claude-md
/claude-md global               # also include the symlinked template (warns about blast radius)
```
