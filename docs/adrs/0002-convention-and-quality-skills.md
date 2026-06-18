# 0002 — Convention & quality skills (design-patterns, claude-md-doctor, improvement-advisor)

**Status:** Accepted
**Date:** 2026-06-18
**Deciders:** Peter Žnuderl

## Context

The audit chain (`repo-audit` → `repo-verifier` → `docs-author`) tells you what a codebase contains and whether it matches its closed issues, and regenerates human docs. Three gaps remained for a solo dev / small team:

1. **Conventions live only in people's heads.** A repo has a house style — how errors are handled, how state is managed, how modules expose their surface — but it's rarely written down. New contributors and agents re-derive it (often wrongly) every time.
2. **CLAUDE.md drifts from reality.** Agents "don't follow our conventions" mostly because the conventions were never put in CLAUDE.md, or the CLAUDE.md says something the code stopped doing. Nothing in the marketplace checked agent guidance against the actual code.
3. **Findings aren't a plan.** `repo-audit` describes findings but never prioritizes or prescribes. Turning an audit into "what should I do next" was manual.

## Decision

Add three skills and one enhancement, grouped as a **Convention & quality** strand of the existing backward-looking chain:

- `design-patterns` — read-only catalog of patterns in use. Source of truth (`patterns.json`) for the other two.
- `claude-md-doctor` — repairs CLAUDE.md against `patterns.json` / `audit.json`. Diff-gated, preservation-aware, never auto-commits, opt-in for the global template.
- `improvement-advisor` — prioritizes findings into a sequenced backlog; opt-in handoff to `github-issue-creator`.
- `docs-author` (enhanced) — consumes `patterns.json` to enrich `docs/architecture.md`, rather than introducing a redundant architecture-doc skill.

`patterns.json` and `improvements.json` carry `schema_version: "1.0"`, consistent with the rest of the chain.

## Consequences

**Upside.**

- The "document the house style → fix the agent guidance → plan the cleanup" loop is now first-class and reuses the audit's `package map` and findings.
- No new manifest shape, MCP, or breaking schema change — purely additive (0.1.0 → 0.2.0).
- Architecture documentation got richer by extending `docs-author` instead of duplicating it, keeping a single doc-generation surface.

**Downside.**

- Skill count is now 20. ADR 0001 set ~25 as the threshold for revisiting the single-plugin shape; we are closer to it. The likely future split (`solo-core`, `solo-planning`) would put this strand with the audit chain in core.
- Three skills share inputs; a stale `audit.json` now affects more outputs. Each skill warns on staleness, but the coupling is real.
- `claude-md-doctor` is the first skill that writes to CLAUDE.md. The diff-gate + draft-first + global-opt-in rules keep it safe, but it widens the set of files the marketplace will modify.

## Alternatives considered

- **A standalone `architecture-doc` skill** — rejected as redundant with `docs-author` and `repo-audit`'s cross-package map. Enriching `docs-author` delivers the same value without a second generator that could drift.
- **Fold pattern detection into `repo-audit`** — rejected; audit is already heavyweight, and patterns are useful on their own (e.g. before writing a CLAUDE.md) without a full audit. Kept separate, with `audit.json` as an optional accelerant.
- **Let `improvement-advisor` file issues directly** — rejected; `github-issue-creator` already owns issue creation with its own gates. Clean handoff via `improvements.json` avoids duplicating that logic.
