# 0003 — Codebase-understanding skills & continuous context

**Status:** Accepted
**Date:** 2026-06-18
**Deciders:** Peter Žnuderl

## Context

An evaluation of the marketplace's document generation found that the context given to coding agents was strong at *describing repo state on demand* but had three structural gaps for a *growing* codebase:

1. **Snapshot, not continuous** — docs are regenerated on demand and drift silently between runs, with no signal when they go stale.
2. **Global, not local** — the richest artifacts (`architecture.md`, `patterns.md`) are repo-wide; an agent editing one file gets little context where it works.
3. **Structure, not behavior** — docs describe how packages are arranged, not how requests actually flow through them.

The 0.2.0 Convention & quality strand addressed conventions. This decision addresses depth-of-understanding and keeping context fresh.

## Decision

Add four skills and enhance four existing ones, grouped as a **Codebase understanding** strand:

- `code-map` (new) — durable, queryable symbol + reference index, incrementally refreshable. The structural backbone the other skills query.
- `flow-docs` (new) — behavioral sequence docs for critical paths; the layer `architecture.md` lacks.
- `domain-glossary` (new) — ubiquitous language with synonym/collision detection.
- `package-context-author` (new) — local, delta-only agent context per package.
- `documentation-check` (enhanced) — git-wired freshness ledger for continuous drift detection.
- `github-issue-implementer` (enhanced) — doc-impact phase so changes close the doc loop.
- `scaffold-repo-meta` (enhanced) — `context_index` so agents navigate generated context.
- `repo-audit` (enhanced) — append-only quality-baseline trend with regression alarms.

New JSON outputs (`code-map.json`, `glossary.json`, `quality-history.json`, the freshness ledger) carry `schema_version: "1.0"`, consistent with the chain. The shift in emphasis is from *pull* (run a skill to get context) to adding *push* (continuous signals as code changes) — the freshness ledger and the implementer's doc-impact phase are the first push mechanisms.

## Consequences

**Upside.**

- Agents get local context (`package-context-author`), navigable structure (`code-map`), behavioral understanding (`flow-docs`), and consistent vocabulary (`domain-glossary`) — the four things the evaluation flagged as missing.
- Context is kept current automatically: drift is detected (`documentation-check` ledger) and repaired at the moment of change (`implementer` doc-impact), rather than depending on someone re-running generators.
- All additive — no breaking schema or manifest change (0.2.0 → 0.3.0). New skills reuse `audit.json` / `patterns.json`.

**Downside.**

- Skill count is now 24 — past the "~25, revisit the single-plugin shape" threshold from ADR 0001. The next addition should seriously weigh a `solo-core` / `solo-planning` split; this strand would sit in core with the audit chain.
- More artifacts means more to keep fresh. The freshness ledger mitigates this but is itself state that can rot if no skill stamps it; ownership is documented per-skill but relies on the skills calling the update.
- `code-map` and `flow-docs` are heavier to run on large repos; both sample and warn, but the marketplace's "never silently sample" rule now spans more skills.
- Cross-skill coupling increased: `flow-docs` and `domain-glossary` are best with `code-map`; `improvement-advisor` now also reads `quality-history.json`. Each degrades gracefully when an input is missing, but the dependency web is wider.

## Alternatives considered

- **Fold the code map into `repo-audit`** — rejected; the audit is a point-in-time judgement, while the code map must persist and refresh incrementally. Different lifecycles; kept separate, with the audit reusing the map when present.
- **One combined "understanding" skill** — rejected; symbol indexing, flow tracing, glossary extraction, and per-package context have distinct inputs, outputs, and cadences. Bundling would hurt triggering and reuse.
- **A real static-analysis/LSP backend for `code-map`** — deferred; would give precise resolution but adds a heavy dependency and language-server setup, against the solo "minimum onboarding friction" principle. The heuristic extractor with explicit low-confidence marking is the pragmatic first cut; revisit if accuracy proves insufficient.
- **Auto-updating docs on every commit (a hook that regenerates)** — rejected as too aggressive and not diff-gated. The chosen design reports drift and *offers* repair, keeping the human/agent in the loop per the marketplace's no-auto-write ethos.
