# CHANGELOG

This marketplace ships one plugin (`solo-claude`). Tracked here with semantic versioning. When making changes, bump the version in **both** `marketplace.json` and the plugin's `plugin.json`.

## solo-claude

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
