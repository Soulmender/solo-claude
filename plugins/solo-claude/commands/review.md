---
description: Code review of a diff against solo-claude conventions, with optional GitHub PR posting
argument-hint: "<branch, range, PR URL, focus, or 'quick'/'full' mode>"
---

# /review

Wraps `code-review` with sensible defaults.

```
/review                              # current branch vs default branch (main)
/review since v1.3.0
/review https://github.com/owner/repo/pull/42
/review quick                        # security + correctness only
/review focus security
```
