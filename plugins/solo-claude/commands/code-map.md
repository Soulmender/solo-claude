---
description: Build/refresh a queryable code map (symbols + who-calls-what)
argument-hint: "<empty for full build, 'refresh' for incremental, or a package>"
---

# /code-map

Wraps `code-map`. Writes `docs/code-map.json` + `docs/code-map.md`. Use `refresh` to update only changed modules since the last build.

```
/code-map
/code-map refresh
/code-map packages/api
```
