---
name: domain-glossary
description: Extract and document the domain vocabulary of a codebase — the ubiquitous language — so agents and contributors use the same terms the same way. Mines entities, value objects, statuses, and key nouns from data models, schemas, types, and database tables, defines each from how it's actually used, and flags synonyms and collisions where one concept has several names or one name means several things. Writes docs/glossary.md plus glossary.json (schema_version 1.0). Honors .repo-meta.yaml. Diff-gated, draft-first, preservation-aware. Use to prevent naming drift as a codebase grows, to onboard to an unfamiliar domain, or before naming new entities. Trigger phrases include "build a glossary", "document our domain terms", "what a domain term means in this codebase", "find naming collisions", "extract the ubiquitous language".
---

# domain-glossary

Pins the words. As a codebase grows and more hands (and agents) touch it, the same concept drifts into three names and one name comes to mean three things. A shared glossary keeps naming consistent — a quiet but real lever on code quality and comprehension.

## What it produces

- `docs/glossary.md` — alphabetized domain terms, each with a definition, where it lives, and synonyms/collisions.
- `.claudedocs/glossary/glossary.json` — `schema_version: "1.0"`, consumable by `claude-md-doctor` (to encode naming rules) and `code-review` (to flag off-vocabulary naming).

Drafts in `.claudedocs/glossary/`; promotion user-gated.

## Inputs

- `audit.json` — the domain-model section names the core entities.
- `code-map.json` — types/classes and their usage spread (which packages use a term).
- Source: data models, schema definitions (Zod/Pydantic/Prisma/SQL DDL), TypeScript types, enums/status unions, DB migrations.

## Phase 1 — Harvest candidate terms

Collect nouns that carry domain meaning: entity/model names, value objects, enums and status values, event names, and recurring identifiers in types and table names. Exclude framework/infra terms (Request, Response, Repository-the-pattern) unless the domain redefines them. Weight by spread — a term used across many packages matters more than a local one.

## Phase 2 — Define from usage

For each term, write the definition from how the code actually uses it — fields, relationships, lifecycle/status transitions, and the operations performed on it — not from a guessed dictionary meaning. Cite anchors (the type/model definition). Capture the term's lifecycle if it's a stateful entity (e.g. `Order: draft → placed → fulfilled → cancelled`).

## Phase 3 — Detect synonyms and collisions

The highest-value output:

- **Synonyms** — distinct names for one concept (`User` vs `Account` vs `Member`). List them; recommend a canonical term (don't rename code — that's `improvement-advisor` territory).
- **Collisions** — one name, several meanings across packages (`Status` meaning order-status here and payment-status there). Flag with the conflicting anchors; these cause real bugs.
- **Drift** — a term whose definition diverges between packages.

## Phase 4 — Write artifacts

### glossary.md

```
# Domain glossary — <repo_name>

**Generated:** <date> · **Commit:** <sha>

## Terms
### Order
Definition (from usage). Lifecycle: draft → placed → fulfilled → cancelled.
Defined: packages/shared/src/models/order.ts:10. Used in: api, web.
Synonyms: none. Collisions: none.

(repeat, alphabetized)

## Synonyms to reconcile
- User / Account / Member → recommend canonical: **User**

## Collisions (one name, several meanings)
- Status — order vs payment. Anchors: ...
```

### glossary.json

```json
{
  "schema_version": "1.0",
  "repo_name": "...",
  "generated_at": "2026-06-18",
  "commit": "...",
  "terms": [
    {
      "term": "Order",
      "definition": "A customer's request to purchase, persisted after payment authorizes.",
      "kind": "entity",
      "lifecycle": ["draft", "placed", "fulfilled", "cancelled"],
      "defined_at": "packages/shared/src/models/order.ts:10",
      "used_in": ["@my-app/api", "@my-app/web"],
      "synonyms": [],
      "collisions": []
    }
  ],
  "synonyms_to_reconcile": [
    {"concept": "user-account", "names": ["User", "Account", "Member"], "recommended": "User"}
  ],
  "collisions": [
    {"name": "Status", "meanings": [
      {"sense": "order status", "anchor": "..."},
      {"sense": "payment status", "anchor": "..."}
    ]}
  ]
}
```

## Phase 5 — Diff, gate, hand off

Diff against any existing glossary, preserved regions untouched. On acceptance write drafts and offer promotion. Offer hand-offs: `claude-md-doctor` to encode a "use canonical term X" rule, and `improvement-advisor` to schedule any rename worth doing.

## Edge cases and rules

- **Thin domain** (mostly CRUD, few real concepts). Produce a short glossary; don't inflate with framework nouns.
- **No models/schemas** (dynamically typed, no declarations). Harvest from variable/table/function naming; lower confidence; say so.
- **Polyglot repo.** One glossary; note where a term's representation differs by language (a `snake_case` table vs a `PascalCase` type for the same concept — that's a synonym, not two terms).
- **Term defined inconsistently across packages.** Report as drift with all anchors; recommend one canonical definition; never silently pick.
- **Sensitive content.** Definitions and examples use obviously-fake values; never embed real customer data.

## Distinguishing from other skills

- **vs. `design-patterns`** — that documents *how* code is written; this documents *what the domain words mean*.
- **vs. `code-map`** — that indexes symbols structurally; this defines the subset that are domain concepts and reconciles their names.
- **vs. `docs-author`** — architecture.md may reference the glossary, but the vocabulary catalog is this skill's job.

## What this skill never does

- Never renames code or modifies source — recommends canonical terms; renames route to `improvement-advisor`.
- Never invents a definition not grounded in usage; thin evidence is marked low confidence.
- Never silently resolves a collision or synonym — surfaces the choice.
- Never auto-promotes drafts.
