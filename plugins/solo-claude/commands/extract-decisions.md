---
description: Extract decisions from a PR, issue, or ADR
argument-hint: "<PR URL, issue ref, or ADR path>"
---

# /extract-decisions

Wraps `decision-ledger` in extract mode.

```
/extract-decisions https://github.com/owner/repo/pull/42
/extract-decisions #123
/extract-decisions docs/adrs/0007-event-bus.md
```
