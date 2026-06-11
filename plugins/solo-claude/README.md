# solo-claude (plugin)

The single plugin shipped by the `solo-claude` marketplace. See the marketplace-root `README.md` for the overview; this file documents the plugin's contents.

## Skills (17)

**Utility (4):**
- `code-review` — adaptive-depth diff review.
- `release-notes` — from git + closed milestones.
- `documentation-check` — doc-hygiene scan.
- `docs-update` — ad-hoc in-repo Markdown edits with preservation.

**Audit chain (3):**
- `repo-audit` — monorepo-aware full audit.
- `repo-verifier` — audit vs. closed-issue requirements.
- `docs-author` — generate / refresh docs from audit.

**GitHub workflow (5):**
- `github-issue-creator` — structured issue creation.
- `github-issue-implementer` — plan + execute under supervision.
- `workflow-hygiene-check` — issue / PR hygiene digest.
- `scaffold-repo-meta` — generate `.repo-meta.yaml`.
- `decision-ledger` — capture and apply decisions; annotate audits.

**Planning chain (5):**
- `requirements-intake` — vague brief to structured intake.
- `package-context-scan` — intra-monorepo context.
- `solution-design` — candidate architectures, recommendation, ADR flags.
- `delivery-plan` — phased epic + child issues breakdown.
- `plan-new-work` — orchestrator with gates and downstream options.

## Slash commands (14)

`/review`, `/release`, `/docs-check`, `/edit-doc`, `/audit`, `/verify`, `/docs`, `/create-issue`, `/implement`, `/hygiene`, `/scaffold`, `/record-decision`, `/extract-decisions`, `/plan`.

## MCP servers

`github` — wired to the standard GitHub MCP endpoint. Authenticate per machine via `/mcp authenticate github`.

## Conventions this plugin relies on

- **`.repo-meta.yaml`** at the root of each repo using these skills. Schema at the marketplace root `docs/repo-meta-yaml-schema.md`.
- **`.claudedocs/`** for Claude-generated drafts; gitignored by convention. The orchestrator pre-flights `.gitignore`.
- **Conventional Commits** for git history.
- **GitHub label namespaces** — `type:*` and `area:*`.
- **JSON schema versions** — all chain JSONs carry `schema_version: "1.0"`. Downstream skills refuse incompatible majors.

## Modifying skills

See marketplace-root `docs/contributing-skills.md`.
