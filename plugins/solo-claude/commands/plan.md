---
description: Run the end-to-end planning chain on a vague brief
argument-hint: "<brief, issue number, file path, or slug to resume>"
---

# /plan

Wraps `plan-new-work`.

```
/plan We want a reservation expiry timer for incomplete orders
/plan Extract a shared library for our cron retry logic
/plan 123                            # plan from an existing GitHub issue
/plan ./briefs/multi-currency.md
/plan reservation-expiry-timer       # resume an existing slug
```
