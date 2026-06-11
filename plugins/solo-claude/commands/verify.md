---
description: Verify the audit JSON against closed GitHub issues, in-repo specs, or pasted requirements
argument-hint: "<milestone, label filter, spec path, or 'recent'>"
---

# /verify

Wraps `repo-verifier`.

```
/verify milestone v1.4.0
/verify recent                     # closed in last 90 days
/verify docs/spec.md
```
