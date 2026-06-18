---
description: Document how a critical path works end to end (sequence diagram)
argument-hint: "<flow name/description, or empty to pick from candidates>"
---

# /flows

Wraps `flow-docs`. Writes `docs/flows/<slug>.md` with a Mermaid sequence diagram, data shapes, failure behavior, and invariants. Run `code-map` first for best accuracy.

```
/flows
/flows the checkout request lifecycle
/flows auth
```
