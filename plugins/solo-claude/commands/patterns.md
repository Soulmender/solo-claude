---
description: Catalog the design patterns and conventions the repo actually uses
argument-hint: "<repo path, single package name, or empty for cwd>"
---

# /patterns

Wraps `design-patterns`. Output to `docs/patterns.md` + `.claudedocs/patterns/patterns.json`. Reads `audit.json` for a richer scan when present.

```
/patterns
/patterns packages/api          # scope to one package
```
