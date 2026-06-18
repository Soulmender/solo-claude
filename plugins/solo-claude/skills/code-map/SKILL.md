---
name: code-map
description: Build a persistent, queryable map of a codebase — modules, their exported symbols, and who-calls-what — so a coding agent can answer "where is X defined, what depends on it, what is safe to change" without re-reading the repo each time. Walks the monorepo (honoring .repo-meta.yaml layout), extracts public symbols per module, resolves internal references into a call/import graph, and flags high-fan-in hotspots and public surfaces. Writes docs/code-map.json (machine, schema_version 1.0) plus a human docs/code-map.md overview, and supports incremental refresh against changed files so it stays cheap as the codebase grows. Use to give agents a navigable index of a growing codebase, to find blast radius before a change, or to locate a symbol's definition and callers. Trigger phrases include "build a code map", "index the codebase", "what depends on this function", "where is X defined", "map the symbols and references", "refresh the code map".
---

# code-map

A navigable index of the codebase that persists between sessions. `repo-audit` builds a dependency graph once and narrates it; this skill keeps a queryable symbol-and-reference map an agent consults on demand — the backbone for understanding a growing codebase and computing blast radius before editing.

## What it produces

- `docs/code-map.json` — machine-readable index, `schema_version: "1.0"`. The primary artifact; agents read it directly.
- `docs/code-map.md` — human overview: module list, public surfaces, hotspots, and how to read the JSON.

Drafts in `.claudedocs/code-map/`; promotion to `docs/` user-gated. The JSON records the `commit` it was built at so staleness is detectable.

## Inputs

- `.repo-meta.yaml` — layout, packages, roles.
- `audit.json` if present — reuse its package map and dependency graph rather than recomputing.
- The source tree. Language-aware extraction (TS/JS, Python at minimum; degrade gracefully for others).

## Phase 1 — Enumerate modules

Walk packages from `.repo-meta.yaml` (or infer). Within each, list modules (files or logical units). Record `path`, owning package, and role. Exclude generated code (banner detection), build output, and vendored deps.

## Phase 2 — Extract symbols

Per module, extract the **public** symbols — exported functions, classes, types, constants, components, hooks; for apps also routes, CLI commands, and event handlers. For each symbol record: name, kind, signature (best-effort), the `path:line` anchor, and a one-line purpose when cheaply inferable from a docstring or name. Internal/private symbols are counted but not individually indexed unless they're call-graph targets.

## Phase 3 — Resolve references

Build the reference graph: for each public symbol, find call sites and imports across the repo. Resolve within-package and cross-package edges. Where static resolution is ambiguous (dynamic dispatch, re-exports, barrel files), record the edge as `confidence: low` rather than dropping or inventing it. Roll edges up to a module-level and package-level import graph (reuse `audit.json`'s if compatible).

## Phase 4 — Derive signals

- **fan-in** per symbol/module — how many distinct callers. High fan-in = high blast radius; an agent changing it must be careful.
- **public surface** per package — the exported set other packages actually consume (vs. exported-but-unused, a cleanup candidate to hand to `improvement-advisor`).
- **hotspots** — top fan-in symbols and modules.
- **orphans** — exported but zero internal/external references (dead-surface candidates).

## Phase 5 — Write artifacts

### code-map.json structure

```json
{
  "schema_version": "1.0",
  "repo_name": "...",
  "generated_at": "2026-06-18",
  "commit": "...",
  "layout": "pnpm-workspaces",
  "modules": [
    {
      "path": "packages/api/src/orders.ts",
      "package": "@my-app/api",
      "symbols": [
        {
          "name": "createOrder",
          "kind": "function",
          "signature": "createOrder(input: OrderInput): Promise<Result<Order>>",
          "anchor": "packages/api/src/orders.ts:42",
          "purpose": "Validate and persist a new order",
          "fan_in": 7,
          "called_by": ["packages/web/src/checkout.ts:88", "packages/api/src/routes/orders.ts:15"]
        }
      ]
    }
  ],
  "package_graph": { "@my-app/api": ["@my-app/shared"] },
  "hotspots": [{"symbol": "createOrder", "anchor": "...", "fan_in": 7}],
  "orphans": [{"symbol": "legacyExport", "anchor": "..."}]
}
```

### code-map.md

Module-by-package listing, the public surface per package, a hotspots table (top fan-in), an orphans list, and a short "how to query this" note for agents (find a definition by symbol name; find callers via `called_by`; gauge blast radius via `fan_in`).

## Phase 6 — Incremental refresh

If a `code-map.json` exists, support a fast refresh: diff `git` since its `commit`, re-extract only changed modules and any module whose edges point into them, and update in place. Full rebuilds only on request or when the layout changed. This is what keeps the map affordable as the codebase grows. Always update `commit` and `generated_at`.

## Edge cases and rules

- **Language the extractor doesn't fully support.** Index files and exports by heuristic; mark `confidence: low`; never fabricate signatures.
- **Barrel files / re-exports.** Follow one hop to the real definition; record the barrel as a pass-through, not the owner.
- **Dynamic / reflective calls.** Can't resolve statically — record what's known, mark low confidence, don't guess targets.
- **Very large repo.** Index public surface fully; sample private call-graph depth and say so. Offer to scope to named packages.
- **No audit.** Run standalone; it just costs more to build the graph from scratch.
- **Monorepo with cycles.** Reflect them faithfully; flag, and cross-link to the audit's cycle finding rather than re-judging.

## Distinguishing from other skills

- **vs. `repo-audit`** — audit *judges health* (findings, severity) at a point in time. This skill *maps structure* and is meant to persist and be queried/refreshed. Audit narrates; code-map indexes.
- **vs. `design-patterns`** — that catalogs *how* code is written (idioms). This catalogs *what exists and what references it* (symbols, edges).
- **vs. `package-context-scan`** — that gathers planning context for one piece of new work, ephemeral. This is a durable repo-wide index.

## What this skill never does

- Never modifies source. Indexes only.
- Never invents a symbol, signature, or edge it couldn't resolve — unresolved edges are marked low-confidence, not omitted silently or fabricated.
- Never grades code quality (fan-in is a signal, not a verdict — judgment belongs to `code-review` / `improvement-advisor`).
- Never auto-promotes drafts.
