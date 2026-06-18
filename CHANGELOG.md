# CHANGELOG

This marketplace ships one plugin (`solo-claude`). Tracked here with semantic versioning. When making changes, bump the version in **both** `marketplace.json` and the plugin's `plugin.json`.

## solo-claude

### 0.3.0 — Codebase-understanding skills & continuous context

Additive minor. No breaking changes; existing skills and JSON schemas are unchanged (new outputs are added, none renamed or removed).

**New skills (4):**

- `code-map` — persistent, queryable index of modules, exported symbols, and who-calls-what, with fan-in/blast-radius signals. Writes `docs/code-map.{json,md}` (`schema_version: "1.0"`); supports incremental refresh against changed files so it stays cheap as the codebase grows.
- `flow-docs` — the behavioral layer `architecture.md` lacks: end-to-end sequence/flow docs for critical paths (request lifecycle, auth, pipelines) with Mermaid sequence diagrams, data shapes, failure behavior, and invariants. Writes `docs/flows/*.md`.
- `domain-glossary` — extracts the domain vocabulary (ubiquitous language) from models/schemas/types, defines each term from usage, and flags synonyms and name collisions. Writes `docs/glossary.{md,json}` (`schema_version: "1.0"`).
- `package-context-author` — generates scoped, agent-facing context per package (`packages/<pkg>/CLAUDE.md`): local role, public surface, blast radius, conventions, gotchas, and "do not do X here". Delta-only against the root CLAUDE.md; diff-gated, preservation-aware.

**Enhanced:**

- `documentation-check` — adds a git-wired **freshness ledger** (`.claudedocs/doc-checks/freshness-ledger.json`) that records the commit each doc was last verified against and reports drift continuously, so a hook or schedule can answer "what did my change make stale". Reports only; never blocks a commit.
- `github-issue-implementer` — adds a **doc-impact** closing phase: after a change, flags which living docs the diff invalidated and offers to refresh them via the owning skills. Suggestion gate; never automatic.
- `scaffold-repo-meta` — maintains a `context_index` block (which context artifacts exist + when to read each), so agents navigate generated context instead of rescanning.
- `repo-audit` — records an append-only **quality baseline** (`docs/audit/quality-history.json`): coverage, finding counts, dependency health, hotspots per run, with regression/improvement deltas surfaced in the report. `improvement-advisor` reads the trend to prioritize against worsening metrics.

**Schema doc:** `docs/repo-meta-yaml-schema.md` documents the new `context_index` block and the `audit.quality_history` path.

**New slash commands (4):** `/code-map`, `/flows`, `/glossary`, `/pkg-context`.

Skill count 20 → 24; slash commands 17 → 21. See ADR `docs/adrs/0003-codebase-understanding-skills.md`.

### 0.2.0 — Convention & quality skills

Additive minor. No breaking changes; all existing skills and JSON schemas are unchanged.

**New skills (3):**

- `design-patterns` — catalogs the design patterns and conventions a repo actually follows (error handling, state, DI, module boundaries, data access, testing, naming, and more), classifies each by adoption / intentionality / confidence, and flags per-package divergences. Writes `docs/patterns.md` + `patterns.json` (`schema_version: "1.0"`). Reads `audit.json` when present.
- `claude-md-doctor` — audits CLAUDE.md files (root, per-package, and optionally the symlinked global template) against the conventions the code actually uses, surfacing gaps, contradictions, stale references, and vague rules. Proposes diff-gated, preservation-aware repairs. Never auto-commits; never edits the global template without explicit opt-in.
- `improvement-advisor` — turns `audit.json` / `patterns.json` / `verification.json` into a prioritized improvement backlog scored by impact × effort, tiered into quick-wins / strategic / long-haul and sequenced by dependency. Writes `docs/improvements.md` + `improvements.json` (`schema_version: "1.0"`). Optional opt-in handoff to `github-issue-creator`. Consults `.decisions.yaml`.

**Enhanced:**

- `docs-author` — now consumes `patterns.json` when available and enriches `docs/architecture.md` with a "Design patterns in use" section and pattern-aware diagrams. No change to its other outputs or behavior when `patterns.json` is absent.

**New slash commands (3):** `/patterns`, `/claude-md`, `/improve`.

Skill count 17 → 20; slash commands 14 → 17. See ADR `docs/adrs/0002-convention-and-quality-skills.md`.

### 0.1.0 — Initial release

- Marketplace and plugin scaffolded for solo developers / small teams on a monorepo, with GitHub Issues and in-repo docs.
- Ships the GitHub MCP endpoint (`mcpServers.github`).
- CLAUDE.md template at `claude-md-templates/solo-CLAUDE.md` (symlink to `~/.claude/CLAUDE.md` at install time).
- Canonical `.repo-meta.yaml` schema at `docs/repo-meta-yaml-schema.md`.

**Skills shipped:**

Utility:
- `code-review` — adaptive-depth review against solo standards, optional PR posting.
- `release-notes` — from git history and closed milestones; GitHub-release-body format.
- `documentation-check` — lightweight doc-hygiene; honors `.repo-meta.yaml` docs layout.
- `docs-update` — edit one in-repo Markdown file, preservation-aware.

Audit chain:
- `repo-audit` — monorepo-aware end-to-end audit. Per-package + cross-package map. Consults `.decisions.yaml`.
- `repo-verifier` — checks audit against closed GitHub issues, in-repo specs, or pasted requirements.
- `docs-author` — regenerates README, `docs/architecture.md`, `docs/runbook.md`, per-package READMEs, ADR stubs.

GitHub workflow:
- `github-issue-creator` — structured issues with label + milestone discipline; epic decomposition.
- `github-issue-implementer` — read issue, plan, execute under supervision. Hard plan + commit gates.
- `workflow-hygiene-check` — digest of unhealthy issue / PR / milestone states.
- `scaffold-repo-meta` — auto-discovers GitHub coordinates, monorepo layout, docs layout; generates `.repo-meta.yaml`.
- `decision-ledger` — capture, query, apply decisions; extract from PR / issue / ADR.

Planning chain:
- `requirements-intake` — classifies new-package / new-module / library-extension / package-expansion; up to 3 rounds of clarifying questions.
- `package-context-scan` — intra-monorepo context for the planned work.
- `solution-design` — 2–3 candidate architectures, pressure-tested, recommendation with rationale, ADR-worthy decisions flagged.
- `delivery-plan` — P0/P1/P2 cuts; parent issue + child issues honoring GitHub conventions.
- `plan-new-work` — orchestrator with gates between phases. Pre-flights `.gitignore`. Optional downstream: GitHub issues, ADR drafts, RFC under `docs/proposals/`, scaffold-repo-meta, decision-ledger.

**Slash commands:**

`/review`, `/release`, `/docs-check`, `/edit-doc`, `/audit`, `/verify`, `/docs`, `/create-issue`, `/implement`, `/hygiene`, `/scaffold`, `/record-decision`, `/extract-decisions`, `/plan`.

All JSON outputs carry `schema_version: "1.0"`. Downstream skills refuse to consume incompatible versions.
