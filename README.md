# solo-claude

A Claude Code marketplace for solo developers and small teams working in a monorepo, tracking issues on GitHub, and writing docs directly in the repo.

One plugin (`solo-claude`). Seventeen skills. Fourteen slash commands. One CLAUDE.md template. Opinionated defaults so you don't have to configure your way in.

## What's inside

| Category | Skills |
|---|---|
| **Utility** | `code-review`, `release-notes`, `documentation-check`, `docs-update` |
| **Audit chain** | `repo-audit`, `repo-verifier`, `docs-author` |
| **GitHub workflow** | `github-issue-creator`, `github-issue-implementer`, `workflow-hygiene-check`, `scaffold-repo-meta`, `decision-ledger` |
| **Planning chain** | `requirements-intake`, `package-context-scan`, `solution-design`, `delivery-plan`, `plan-new-work` (orchestrator) |

Each skill is documented in its own SKILL.md under `plugins/solo-claude/skills/`.

## Quickstart

```bash
# Add the marketplace once
/plugin marketplace add https://github.com/<your-user>/solo-claude
#   (or file:// if you cloned locally)

# Install the plugin
/plugin install solo-claude@solo-claude

# Verify
/plugin list
```

After install, reload Claude Code or run `/reload-plugins`.

## Symlink the team-wide CLAUDE.md (one-time setup)

`solo-claude` ships a CLAUDE.md template at `claude-md-templates/solo-CLAUDE.md`. Link it to `~/.claude/CLAUDE.md` so it activates on every Claude Code session:

**macOS / Linux:**
```bash
ln -sf <plugin-install-path>/claude-md-templates/solo-CLAUDE.md ~/.claude/CLAUDE.md
```

**PowerShell (Windows, admin):**
```powershell
New-Item -ItemType SymbolicLink -Path "$env:USERPROFILE/.claude/CLAUDE.md" -Target "<plugin-install-path>/claude-md-templates/solo-CLAUDE.md"
```

Future `/plugin update solo-claude` refreshes the template; the symlink follows automatically.

## Authenticate the GitHub MCP

The plugin declares the GitHub MCP. After install, authenticate per machine:

```
/mcp authenticate github
```

Skills that touch GitHub Issues / PRs will fail with a clear auth error if you skip this step.

## Onboarding a repo

In each repo where you want solo-claude skills to be smart about context, run:

```
/scaffold
```

This generates `.repo-meta.yaml` by inspecting your git remote, monorepo layout, docs folder, tags, and labels. It prompts for only the things it can't infer. See `docs/repo-meta-yaml-schema.md` for the full schema.

After scaffolding, the following land for free:

- `.repo-meta.yaml` at the root.
- `.gitignore` updated to include `.claudedocs/` (Claude's draft folder).
- A suggestion of what to run next.

## The two chains

### Backward-looking (audit existing code)

```
repo-audit          → docs/audit/audit.{md,json}
       ↓
repo-verifier       → docs/audit/verification.{md,json}
       ↓
docs-author         → README.md + docs/architecture.md + docs/runbook.md + docs/adrs/ stubs
```

### Forward-looking (plan new work)

```
requirements-intake  → intake.{md,json}
       ↓
package-context-scan → package-context.{md,json}
       ↓
solution-design      → design.{md,json}
       ↓
delivery-plan        → delivery-plan.{md,json}
       ↓
(opt-in) github-issue-creator + ADR drafts + RFC under docs/proposals/ + scaffold-repo-meta + decision-ledger
```

Run the forward chain end-to-end via `/plan <brief>`. Each step in both chains is rerunnable in isolation.

## Slash commands

| Command | What it does |
|---|---|
| `/review` | Code review of a diff |
| `/release` | Release notes from git + closed milestones |
| `/docs-check` | Doc-hygiene scan |
| `/edit-doc` | Edit one in-repo Markdown file |
| `/audit` | Full repo audit |
| `/verify` | Verify the audit against closed issues / specs |
| `/docs` | Generate README + docs/ + ADR stubs from the audit |
| `/create-issue` | Create one or more GitHub issues |
| `/implement` | Plan + execute an issue under supervision |
| `/hygiene` | Workflow hygiene digest |
| `/scaffold` | Generate `.repo-meta.yaml` |
| `/record-decision` | Record a decision into `.decisions.yaml` |
| `/extract-decisions` | Extract decisions from a PR / issue / ADR |
| `/plan` | End-to-end planning chain |

## Conventions

**`.repo-meta.yaml`** at each repo's root pins GitHub coordinates, monorepo layout, docs convention, label and milestone conventions, and audit artifact paths. Canonical schema: [`docs/repo-meta-yaml-schema.md`](docs/repo-meta-yaml-schema.md).

**`.claudedocs/`** is for Claude-generated drafts (audit reports, planning artifacts, scratch). It's gitignored by convention; the orchestrator's pre-flight checks this and offers to add the entry if missing.

**`.decisions.yaml`** at the repo root captures the team's decisions about findings. The audit and verifier consult it so previously-recorded decisions show up as annotations (never as suppressions).

**Audit and verification artifacts** land at `docs/audit/*` by default. Paths are configurable in `.repo-meta.yaml`'s `audit:` block.

**Conventional Commits** for git history. Drives `release-notes` classification.

**Type and area labels on GitHub Issues** — `type:feature`, `type:bug`, `type:task`, `type:chore`, `type:docs` and `area:<package>`. Configurable in `.repo-meta.yaml`'s `github.labels`.

## How updates work

```bash
/plugin marketplace update      # refresh the catalog
/plugin update solo-claude
/reload-plugins
```

Breaking changes signal via major version bumps in `marketplace.json` and `plugin.json`. The audit/verifier/delivery-plan JSONs carry their own `schema_version`; downstream skills refuse to consume incompatible versions.

## For maintainers

If you're maintaining this marketplace:

- Bump the affected plugin version in both `marketplace.json` and `plugin.json` (semver).
- Add a CHANGELOG entry.
- See `docs/contributing-skills.md` for the per-skill workflow (file layout, frontmatter validation, schema bump rules).

## Documentation

- [`docs/repo-meta-yaml-schema.md`](docs/repo-meta-yaml-schema.md) — canonical schema for the per-repo metadata file.
- [`docs/onboarding.md`](docs/onboarding.md) — day-one setup for a new machine / new repo.
- [`docs/contributing-skills.md`](docs/contributing-skills.md) — writing or modifying a skill.
- [`docs/adrs/`](docs/adrs/) — architecture decisions for the marketplace itself.

## License

MIT. See `LICENSE`.
