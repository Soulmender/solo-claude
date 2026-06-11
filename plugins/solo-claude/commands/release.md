---
description: Generate release notes from git history and closed GitHub milestones / issues
argument-hint: "<version label, git range, milestone, or audience hint>"
---

# /release

Wraps `release-notes` with sensible defaults.

```
/release                             # since last tag
/release v1.4.0                      # since previous tag, label as v1.4.0
/release since v1.2.0
/release for stakeholders            # exec-brief format
```
