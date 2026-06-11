---
name: package-context-scan
description: Gather intra-monorepo context for the planned work — which existing packages it'll touch, what each does, how they interconnect. Auto-discovers packages from .repo-meta.yaml or the manifest; reads each package's manifest, README, and CLAUDE.md; pulls the audit JSON for each package if present; reads the existing in-monorepo dependency graph. Synthesizes an integration map showing which packages own which capabilities and where the planned work would land. Surfaces gaps (package missing README, no recent audit) as explicit findings rather than silent assumptions. Use after requirements-intake. Trigger phrases include "scan the monorepo for this", "what packages does this touch", "check the codebase before we design", "map the impact in this repo". Output is package-context.md plus package-context.json. Schema version 1.0.
---

# package-context-scan

Second phase of `plan-new-work`. The monorepo equivalent of cross-service infra-scan: it stays inside this repo.

## What it produces

- `.claudedocs/<slug>/package-context.md`
- `.claudedocs/<slug>/package-context.json` — `schema_version: "1.0"`.

## Inputs

- `.claudedocs/<slug>/intake.json` — required.
- `.repo-meta.yaml` — used if present; falls back to filesystem inference.

## Phase 1 — Enumerate packages

Read `monorepo.packages` from `.repo-meta.yaml`. If absent, infer from filesystem (same heuristics as `repo-audit`).

For `single-package` repos: the "packages" list is one entry — the repo itself. The scan still runs but the cross-package map collapses to "internal modules" instead.

## Phase 2 — Identify candidate set

Show the user the intake's `work_type` and `classification_details`. Then ask:

> "Which packages do you think this touches? I'll start with those and the obvious neighbors. List by name or pick from this menu."

Show the package list. Let the user pick a subset; default offer is the host package + its direct dependencies + its direct consumers.

This phase exists because in a large monorepo the cross-package fan-out is too much to scan blindly — but you have the user right here, so just ask.

For small repos (under ~5 packages), default to scanning everything and confirm.

## Phase 3 — Per-package context pull

For each candidate, read:

### Manifest
- `package.json` (name, version, main, exports, scripts, deps, devDeps).
- For Python: `pyproject.toml`.

### README
- Per-package `README.md` if present. Extract role, capabilities, public surface.

### CLAUDE.md
- Per-package `CLAUDE.md` if present (repo-level CLAUDE.md is also read as global context).

### Audit JSON
- If `docs/audit/audit.json` exists at the repo level, pull the section for this package. Extract: top capabilities, public API, dependencies, known risks. Note `schema_version`.

### Internal dep graph slice
- Who imports this package, who does it import. Build from the audit if available, otherwise heuristic scan.

If a source is missing, record the gap; don't fail.

## Phase 4 — Synthesize the integration map

For each package the planned work plausibly touches, build a record:

- **package** — name + role.
- **interaction_type** — `consumes-export-from`, `extends-export-of`, `produces-event-for`, `shares-types-with`, `replaces-capability-of`, `coexists-with`.
- **mechanism** — direct import, event bus, shared types, CLI invocation.
- **risk_flag** — true if the planned change would alter a public API, break a downstream consumer, or affect a package the requester doesn't usually touch.

## Phase 5 — Gap report

Status per package:

- `complete` — manifest + README + audit reachable.
- `partial` — manifest reachable, others missing.
- `missing` — package named but not findable (likely user typo).
- `divergent` — README and audit disagree materially (e.g. README says "stable API", audit found instability).

Surface gaps loudly. Solo devs forget which packages have docs.

## Phase 6 — Write artifacts

### package-context.md

```
# Package context: <slug>

**Generated:** <date>
**Packages scanned:** N (M complete, K partial, J missing)
**Monorepo layout:** ...

## Per-package summaries
### <pkg-name>
- Role, stack, capabilities (top 5)
- Public surface
- Internal deps (in monorepo): in / out
- Known risks
- Source status

## Integration map
| Planned capability | Touches | How | Risk |

## Gaps and open questions
```

### package-context.json

```json
{
  "schema_version": "1.0",
  "slug": "...",
  "generated_at": "2026-06-11",
  "monorepo_layout": "pnpm-workspaces",
  "packages": [
    {
      "name": "@my-app/api",
      "path": "packages/api",
      "role": "app",
      "capabilities": ["..."],
      "public_surface": {...},
      "internal_deps": {"in": ["@my-app/web"], "out": ["@my-app/shared"]},
      "known_risks": [...],
      "sources": {
        "manifest": "present",
        "readme": "missing",
        "audit": "present"
      },
      "status": "partial"
    }
  ],
  "integration_map": [
    {
      "planned_capability": "...",
      "package": "@my-app/api",
      "interaction_type": "extends-export-of",
      "mechanism": "new function exported from /handlers/index.ts",
      "risk_flag": false
    }
  ],
  "gaps": [
    {"package": "...", "missing": ["readme"], "blocking_for_design": false}
  ]
}
```

## Edge cases and rules

- **Single-package repo.** Skip cross-package map; render internal-module map instead (folders under `src/`).
- **No audit yet.** Run with manifest + README context only. Recommend `repo-audit` first if the change is large.
- **Audit JSON `schema_version` incompatible.** Warn; use what you can.
- **User names a package that doesn't exist.** Surface as gap; ask whether to skip or rescan with the correction.
- **Massive monorepo (greater than 30 packages).** Don't blindly scan all; force the candidate-set question even if user wanted everything.
- **GitHub MCP not authenticated.** Doesn't matter — this skill stays local.
- **Sensitive content.** Per CLAUDE.md, redact customer identifiers in captured surfaces before writing.

## Distinguishing from other skills

- **vs. `repo-audit`** — that audit is deep and across-everything. This is a planning-context scan focused on what the planned work would touch.
- **vs. `documentation-check`** — that diagnoses doc gaps. This consumes any docs that exist and surfaces gaps as planning blockers.

## What this skill never does

- Never modifies packages.
- Never invents capabilities. Records what sources say; gaps for the rest.
- Never proposes a design. That's `solution-design`.
