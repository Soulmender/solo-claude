---
name: repo-audit
description: Full end-to-end audit of a monorepo or single-package repo. Detects layout (pnpm/yarn workspaces, turborepo, nx, uv, poetry, or single-package) from .repo-meta.yaml or filesystem signals. For monorepos, walks each package and reports per-package plus cross-package integration. For each package, investigates API surface, domain model, business logic, dependencies, build and test setup, and quality signals. Consults .decisions.yaml so previously-recorded team decisions annotate matching findings rather than re-surface as new issues. Records a quality baseline (coverage, finding counts, dependency health, hotspots) appended to a history file so trends show and regressions raise an alarm. Writes Markdown and JSON reports to docs/audit by default. Use when onboarding to an unfamiliar repo, before a major release, or as a quarterly health check. Trigger phrases include "audit this repo", "deep-dive on the monorepo", "review the project end to end", "is code quality getting better or worse".
---

# repo-audit

Comprehensive end-to-end audit. The forward-looking counterpart is the planning chain; this one looks at what's already shipped.

## What it produces

- `docs/audit/audit.md` — human-readable long-form report (or `report_md` from `.repo-meta.yaml`).
- `docs/audit/audit.json` — structured findings with `schema_version: "1.0"`, consumed by `repo-verifier` and `docs-author`.

Drafts land in `.claudedocs/audit/` until the user promotes.

## Phase 1 — Detect repo shape

Read `.repo-meta.yaml`'s `monorepo` block. If missing, infer from filesystem:

- `pnpm-workspace.yaml` → `pnpm-workspaces`
- `package.json` `workspaces` key → `yarn-workspaces`
- `turbo.json` → also `turborepo` overlay (additive on top of workspaces)
- `nx.json` → `nx`
- `pyproject.toml` with `[tool.uv.workspaces]` → `uv-workspaces`
- `pyproject.toml` with `[tool.poetry]` and multiple subdirs → `poetry-monorepo`
- Otherwise → `single-package`

Enumerate packages by reading the relevant manifest. Build a list `[{name, path, role, manifest}]`.

## Phase 2 — Per-package investigation

For each package, run focused passes. Run multiple packages in parallel when the diff between them is meaningful.

### API surface
- Exports listed in `package.json` `exports` / `main` / `module`.
- Public types and functions (TypeScript: read `.d.ts` or top-level exports; JS: heuristics on top-level exports; Python: `__all__` or top-level public symbols).
- For app packages: HTTP routes, CLI commands, GraphQL schemas.

### Domain model
- Data models, schemas, DB tables / migrations, validation shapes.
- Type contracts between packages.

### Business logic
- Where the core decisions live (services, hooks, reducers, handlers).
- Workflows and state machines if any.
- Non-obvious rules.

### Dependencies
- Internal: which packages import which (build the in-monorepo graph).
- External: production deps, dev deps, optional deps. Flag unusual ones (deprecated, abandoned, security-flagged via known CVEs if obvious from version).
- License obligations worth noting (copyleft in production code).

### Build and test setup
- How the package is built (tsup, vite, esbuild, swc, plain `tsc`, `setuptools`, `uv build`).
- How it's tested (Vitest, Jest, Pytest, Playwright, custom).
- CI references — which workflows touch this package.

### Quality signals
- Test coverage (run `--coverage` if cheap; otherwise inspect existing reports).
- Dead code (unused exports).
- TODO / FIXME / XXX / HACK comments — list top 20, surface the categories.
- Recent churn — files changed in last 30 days disproportionate to their size.

## Phase 3 — Cross-package integration map

Build the in-monorepo dependency graph and render:

- Strongly-connected components (cycles between packages — almost always a bug).
- Package fan-in (a package imported everywhere — single point of failure or a healthy shared utility?).
- Package fan-out (a package importing too much — likely doing too many things).
- Public-vs-private boundaries — are libraries' internals leaking into apps?

## Phase 4 — Consult `.decisions.yaml`

If `.decisions.yaml` exists at the repo root, load it. For each finding produced in Phases 2–3, attempt to match against recorded decisions (rule-based, see decision-ledger). Matched findings get an `annotations` entry in the JSON and an inline note in the Markdown — they are NEVER suppressed. Annotations carry decision id, category, rationale summary, source link, and a `stale` flag if the decision is older than 9 months.

## Phase 5 — Headline and severity

Each finding has severity:

- **Blocker** — security issue, data-loss risk, broken contract between packages.
- **Major** — significant architectural risk, dependency cycle, deeply outdated dependency.
- **Minor** — style or quality issue.
- **Nit** — preference.

The headline summarizes: green / yellow / red, with a count per severity, and the top three.

## Phase 6 — Write artifacts

### audit.md structure

```
# Repo audit — <repo_name>

**Generated:** <date>
**Commit:** <sha>
**Layout:** <monorepo-flavor>
**Packages scanned:** N
**Headline:** 🟢 / 🟡 / 🔴 — one-line summary

## Executive summary
## Per-package findings
### <package-name>
- Role, stack, capabilities (top 5)
- Public surface
- External dependencies (notable only)
- Test setup and coverage
- Findings by severity
- Annotated decisions

## Cross-package integration
## Architectural risks
## Open questions
```

### audit.json structure

```json
{
  "schema_version": "1.0",
  "repo_name": "...",
  "generated_at": "2026-06-11",
  "commit": "...",
  "layout": "pnpm-workspaces",
  "packages": [
    {
      "name": "@my-app/web",
      "path": "packages/web",
      "role": "app",
      "capabilities": ["..."],
      "public_surface": {...},
      "internal_deps": ["@my-app/shared"],
      "external_deps": [...],
      "test_setup": {...},
      "findings": [
        {
          "severity": "major",
          "anchor": "packages/web/src/auth.ts:42",
          "category": "...",
          "description": "...",
          "annotations": [...]
        }
      ]
    }
  ],
  "cross_package": {
    "dependency_graph": {...},
    "cycles": [],
    "boundary_violations": [...]
  },
  "headline": "yellow",
  "totals": {"blocker": 0, "major": 3, "minor": 12, "nit": 30}
}
```

## Phase 7 — Quality baseline and trend

A single audit is a snapshot; a growing codebase needs to know whether it's getting better or worse. Capture a compact **baseline** each run and append it to a history file so trends are visible.

Record into `docs/audit/quality-history.json` (`schema_version: "1.0"`) one entry per run:

- `commit`, `generated_at`.
- `coverage` — overall and per-package test coverage (from `--coverage` if cheap, else existing reports; `null` where unknown, never guessed).
- `findings` — counts by severity (`blocker`/`major`/`minor`/`nit`), overall and per package.
- `dependencies` — count outdated / deprecated / security-flagged.
- `hotspots` — count of high-fan-in modules and any dependency cycles (reuse `code-map.json` if present).
- `surface` — public-export count and orphan (dead-surface) count when `code-map.json` is available.

Then compare against the previous entry and surface **deltas** in the report:

- **Regression alarm** — coverage dropped, blocker/major count rose, a new cycle appeared, or dead surface grew. Call these out at the top of the report, loudly.
- **Improvement** — the inverse; note it so cleanup work is visibly rewarded.
- **Flat** — no material change.

History is append-only; never rewrite past entries. If `quality-history.json` is absent, this run establishes the baseline and the report says so (no deltas yet). The trend feeds `improvement-advisor`, which can prioritize against a worsening metric.

## Phase 8 — Update `.repo-meta.yaml`'s audit block

After writing, update the `audit:` block with `last_run`, `commit`, `report_md`, `report_json`, and `quality_history`. Show a diff; write only on confirmation.

## Edge cases and rules

- **Very large repos (greater than 500 source files per package).** Warn at the start; offer to scope down (one package at a time, or skip a named package). Don't silently sample.
- **No `.repo-meta.yaml`.** Run anyway with inferred layout; offer to scaffold one at the end.
- **A package has no tests at all.** Major finding for `app` packages, minor for `tool` packages, nit for `library` packages without behavior.
- **Generated packages** (OpenAPI clients, gRPC stubs). Detect via banner comment; skip detailed pass; note presence only.
- **The repo has private submodules.** Don't try to clone; flag and proceed with what's local.
- **Decision-ledger schema mismatch.** Warn but proceed; annotations may be partial.
- **Single-package repo.** Skip the cross-package phase; render the per-package section without the package selector.

## Distinguishing from other skills

- **vs. `documentation-check`** — this skill is heavyweight architecture + business-logic + integration deep-dive. Doc-check is the 5-minute version. Use doc-check before releases; use audit quarterly or on onboarding.
- **vs. `code-review`** — this skill audits the repo as it stands. Code-review evaluates a specific diff.
- **vs. `repo-verifier`** — this skill maps what the code does. Verifier checks the code against claimed requirements (closed GitHub issues, etc.).

## What this skill never does

- Never modifies code or docs. Reports only.
- Never suppresses a finding even when annotated. Annotations explain; findings remain visible.
- Never auto-promotes drafts from `.claudedocs/` to `docs/audit/`. User-confirmed promotion only.
