# `.repo-meta.yaml` — canonical schema

This file lives at the root of any repo that uses the `solo-claude` skills. It pins per-repo context (GitHub owner/repo, monorepo layout, docs paths, label/milestone conventions, audit artifact paths) so skills don't re-prompt for the same answers every run.

If a skill needs context that's not in `.repo-meta.yaml`, it asks once and offers to add the answer to the file.

## Minimal example

```yaml
repo_name: my-app
github:
  owner: peter
  repo: my-app
monorepo:
  layout: pnpm-workspaces
  packages_root: packages/
docs:
  layout: docs-folder-plus-readme
  docs_root: docs/
audit: {}  # auto-managed by repo-audit / repo-verifier / docs-author
```

## Full example

```yaml
repo_name: my-app
description: A short one-line summary of what this repo is.

github:
  owner: peter
  repo: my-app
  default_branch: main
  labels:
    type:
      - type:feature
      - type:bug
      - type:task
      - type:chore
      - type:docs
    area:
      - area:web
      - area:api
      - area:shared
      - area:infra
    other:
      - good-first-issue
      - blocked
      - help-wanted
  milestone_pattern: "v?\\d+\\.\\d+\\.\\d+"  # regex; release-notes uses this to map milestones to versions

monorepo:
  layout: pnpm-workspaces            # pnpm-workspaces | yarn-workspaces | turborepo | nx | uv-workspaces | poetry-monorepo | single-package
  packages_root: packages/
  packages:                          # optional, useful if package names diverge from folder names
    - name: "@my-app/web"
      path: packages/web
      role: app
    - name: "@my-app/api"
      path: packages/api
      role: app
    - name: "@my-app/shared"
      path: packages/shared
      role: library

docs:
  layout: docs-folder-plus-readme    # docs-folder-plus-readme | mkdocs | docusaurus | readme-only
  docs_root: docs/
  adr_root: docs/adrs/
  rfc_root: docs/proposals/          # optional, used by plan-new-work's Confluence-equivalent step
  preserved_sections:
    - "## Manual notes"
    - "## Maintainer notes"

release:
  versioning: semver                 # semver | calver | none
  changelog_path: CHANGELOG.md
  tag_format: "v{version}"           # used by release-notes

audit:
  last_run: 2026-06-11T10:00:00Z     # auto-managed
  commit: a1b2c3d                    # auto-managed
  report_md: docs/audit/audit.md     # pinned by user, optional
  report_json: docs/audit/audit.json # pinned by user, optional
  verification_md: docs/audit/verification.md
  verification_json: docs/audit/verification.json
  quality_history: docs/audit/quality-history.json  # auto-managed append-only trend

contributors:                        # optional, for ownership context
  - name: Peter
    handle: PeterZnuderl
    role: maintainer

context_index:                       # auto-managed by scaffold-repo-meta and the context skills
  - path: docs/architecture.md
    read_when: "you need the structural map — packages, boundaries, external systems"
    generated_by: docs-author
  - path: docs/patterns.md
    read_when: "you need the house style — how errors, state, modules are done here"
    generated_by: design-patterns
  - path: docs/code-map.json
    read_when: "you need to find a symbol's definition, its callers, or blast radius"
    generated_by: code-map
  - path: docs/flows/
    read_when: "you need to understand how a path behaves end to end before changing it"
    generated_by: flow-docs
  - path: docs/glossary.md
    read_when: "you need the meaning of a domain term or the canonical name to use"
    generated_by: domain-glossary
```

## Field-by-field

### `repo_name` (required)

Used as a fallback identifier when GitHub MCP isn't authenticated. Human-readable, kebab-case.

### `github` (recommended)

`owner` and `repo` are the GitHub coordinates. The GitHub MCP uses them to fetch issues and PRs without asking each run.

`default_branch` defaults to `main` if not set. Skills that need a base for diffing (code-review, release-notes) use this.

`labels` documents what label sets exist in your repo. `github-issue-creator` validates applied labels against this list; `workflow-hygiene-check` flags unlabeled issues against the `type:` and `area:` namespaces specifically. The categorization (`type`, `area`, `other`) is convention only — GitHub doesn't enforce label namespaces.

`milestone_pattern` is a regex. `release-notes` uses it to map closed milestones to release versions. The default `v?\d+\.\d+\.\d+` matches `v1.2.3` and `1.2.3`.

### `monorepo` (required)

`layout` is one of `pnpm-workspaces`, `yarn-workspaces`, `turborepo`, `nx`, `uv-workspaces`, `poetry-monorepo`, `single-package`. The audit skill uses it to know how to walk packages and read manifests.

`packages_root` is the folder containing package subdirectories. Most monorepos use `packages/` (or `apps/` + `packages/` for app/lib split — you can list both in `packages_root` as a list).

`packages` is an optional explicit list. Useful when package names (`@my-app/web`) diverge from folder names (`packages/web`), or when one folder isn't a package. `role` is `app | library | tool | infra` for downstream skills to weight findings.

### `docs` (required)

`layout` selects how the docs-author and docs-update skills behave:

- `docs-folder-plus-readme` — `docs/` for everything beyond `README.md`. Default.
- `mkdocs` — same as above plus `mkdocs.yml` nav updates when adding pages.
- `docusaurus` — same as above plus Docusaurus sidebar updates.
- `readme-only` — no `docs/` folder; everything lives in README sections. docs-author maintains README sections only.

`docs_root` is `docs/` for the first three. Ignored for `readme-only`.

`adr_root` and `rfc_root` are where ADRs and RFCs live. Defaults: `docs/adrs/` and `docs/proposals/`.

`preserved_sections` lists heading texts under which `docs-author` and `docs-update` will not overwrite content. Defaults to `## Manual notes`. Add more if your team has another preservation convention.

### `release` (recommended)

`versioning` is `semver`, `calver`, or `none`. `release-notes` uses this to compute the next version.

`changelog_path` is where the changelog lives. Defaults to `CHANGELOG.md` at the root.

`tag_format` is the template for release tags. Defaults to `v{version}`.

### `audit` (auto-managed)

The audit / verifier / docs-author skills maintain this block. The user pins paths (`report_md`, `report_json`, etc.) on first run; subsequent runs update `last_run` and `commit`. `quality_history` points at the append-only quality-trend file `repo-audit` maintains (coverage, finding counts, dependency health, hotspots per run); `improvement-advisor` reads it to prioritize against worsening metrics.

### `contributors` (optional)

Useful when the repo has more than one contributor and ownership matters for review or planning. Skills surface contributor handles in audit reports and docs when this is set.

### `context_index` (auto-managed)

A map of the agent-facing context artifacts that exist in the repo, each paired with a `read_when` hint and the `generated_by` skill that owns it. `scaffold-repo-meta` seeds it from what's present; the context skills (`docs-author`, `design-patterns`, `code-map`, `flow-docs`, `domain-glossary`, `package-context-author`) update their own entry on promotion. Its purpose is navigation: an agent reads the index to decide which artifact answers its question instead of rescanning the whole repo. Entries pointing at missing files are flagged on the next scaffold run, not silently removed.

## Auto-discovery

If `.repo-meta.yaml` is missing or partial when a skill needs it, the skill offers to run `scaffold-repo-meta`, which:

1. Reads the GitHub remote URL to fill `github.owner` and `github.repo`.
2. Detects the monorepo layout from `pnpm-workspace.yaml`, `package.json` `workspaces`, `turbo.json`, `nx.json`, `pyproject.toml` `[tool.uv.workspaces]`, etc.
3. Detects the docs layout from the presence of `docs/`, `mkdocs.yml`, `docusaurus.config.*`, or absence of any.
4. Detects versioning from `CHANGELOG.md` patterns and existing tags.
5. Asks for the rest.

## Validation

`scaffold-repo-meta --check` (or running any skill with a malformed file) prints schema violations:

- `monorepo.layout` not in the allowed set
- `docs.layout` not in the allowed set
- `github.owner`/`github.repo` missing
- `audit.*` paths pointing at non-existent files (warning, not error)
