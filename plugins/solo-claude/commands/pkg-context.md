---
description: Generate scoped per-package agent context files (local CLAUDE.md)
argument-hint: "<package name/path, or empty to choose interactively>"
---

# /pkg-context

Wraps `package-context-author`. Writes a short `packages/<pkg>/CLAUDE.md` of local conventions, surface, blast radius, and gotchas. Diff-gated, draft-first.

```
/pkg-context
/pkg-context packages/api
```
