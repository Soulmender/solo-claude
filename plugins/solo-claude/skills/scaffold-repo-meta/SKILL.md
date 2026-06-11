---
name: scaffold-repo-meta
description: Generate or extend a .repo-meta.yaml file at the repo root. Auto-discovers the github owner and repo from the git remote, detects monorepo layout from pnpm-workspace.yaml, package.json workspaces, turbo.json, nx.json, pyproject.toml workspaces, or single-package fallback. Detects docs layout from filesystem (docs folder vs README only, mkdocs.yml, docusaurus config). Reads existing tags to infer versioning style. Prompts only for what cannot be inferred. Use when onboarding a repo to solo-claude, when a skill detects a missing .repo-meta.yaml, or for refreshing the file when conventions change. Trigger phrases include "scaffold the repo meta", "set up .repo-meta.yaml", "onboard this repo", "generate the meta file".
---

# scaffold-repo-meta

Generates a sensible `.repo-meta.yaml` with minimum user friction. Reduces onboarding-a-repo to under five minutes.

## What it produces

`.repo-meta.yaml` at the repo root. Optionally a starter `.gitignore` adding `.claudedocs/` if missing.

## Phase 1 — Inspect repo state

If `.repo-meta.yaml` already exists, read it. Tell the user it exists; offer to extend (fill missing fields) or replace (full regeneration with current detected state).

If neither, proceed to fresh generation.

## Phase 2 — Auto-discover

Run these probes; collect findings without prompting yet.

### GitHub coordinates
- Parse `git remote get-url origin`. Extract `owner/repo` from SSH or HTTPS form.
- `default_branch` — query the GitHub MCP if authenticated; otherwise read `git symbolic-ref refs/remotes/origin/HEAD` or fall back to `main`.

### Monorepo layout
- `pnpm-workspace.yaml` present → `pnpm-workspaces`. Parse `packages:` list.
- `package.json` with `workspaces` field → `yarn-workspaces`. Parse the array.
- `turbo.json` → also flag as `turborepo` (overlay; keep the workspaces flavor as primary).
- `nx.json` → `nx`.
- `pyproject.toml` with `[tool.uv.workspaces]` → `uv-workspaces`.
- `pyproject.toml` with `[tool.poetry]` and multiple package subdirs (children with their own `pyproject.toml`) → `poetry-monorepo`.
- None of the above → `single-package`.

For each detected package: capture `{name, path, role}`. Infer `role`:
- Folder name matches `apps/`, `web/`, `api/`, `mobile/` → `app`.
- Folder name matches `lib/`, `packages/<x>`, `libraries/` → `library`.
- Folder name matches `tools/`, `scripts/`, `cli/` → `tool`.
- Otherwise: `library` for shared code, `app` for everything else.

### Docs layout
- `mkdocs.yml` present → `mkdocs`.
- `docusaurus.config.*` present → `docusaurus`.
- `docs/` folder exists → `docs-folder-plus-readme`.
- Only `README.md` → `readme-only`.

### ADR / RFC paths
- `docs/adrs/` exists → set `adr_root`.
- `docs/proposals/` or `docs/rfcs/` exists → set `rfc_root`.

### Versioning
- Latest tag matches `v\d+\.\d+\.\d+` → `semver`, `tag_format: "v{version}"`.
- Latest tag matches `\d+\.\d+\.\d+` → `semver`, `tag_format: "{version}"`.
- Tags match `\d{4}\.\d{2}\.\d+` → `calver`.
- No tags → `none`; recommend setting later.

`changelog_path`: `CHANGELOG.md` if exists at root; otherwise `CHANGELOG.md` as a default.

### Labels
- Pull current labels via GitHub MCP if authenticated. Group by prefix (`type:*`, `area:*`). Render as the `labels` block; flag uncategorized labels in `other`.

### Contributors
- `git shortlog -sne --no-merges | head -10` to get top contributors. Surface, ask the user which to list as collaborators (or skip).

## Phase 3 — Confirm with the user

Render the auto-discovered draft. Highlight inferred fields. Ask:

- "Looks right? Any field to correct?"
- For `single-package`: confirm.
- For `docs.layout: readme-only`: confirm — common but worth checking.
- For `release.versioning: none`: ask whether to leave none, or set a target.

Iterate until the user signs off.

## Phase 4 — Write

Write `.repo-meta.yaml` at repo root. Show a diff if extending an existing file.

After write:

- Verify `.claudedocs/` is gitignored. If not, offer to append to `.gitignore`.
- Suggest a commit: `chore(meta): scaffold .repo-meta.yaml`.

## Phase 5 — Next steps

Tell the user what they can now run:

- `repo-audit` — full audit.
- `documentation-check` — quick doc hygiene.
- `workflow-hygiene-check` — issue tracker hygiene.
- `/plan <brief>` — start a planning chain.

## Edge cases and rules

- **No git remote.** Skip GitHub block; ask the user to set up the remote and re-run, or proceed with `github: {}`.
- **Multiple remotes** (`origin`, `upstream`). Use `origin` by default; ask if ambiguous.
- **Private fork.** The GitHub MCP may not have access. Note this; proceed without label discovery.
- **Detected packages overlap weirdly** (e.g. `apps/web` and `packages/web`). Surface; ask user to disambiguate.
- **`pnpm-workspace.yaml` lists `**` globs.** Resolve to actual directories. Don't write globs into `.repo-meta.yaml`.
- **Mixed-language monorepo.** Honor the primary layout; note the secondary in `description`. (Skills that walk packages will still work per-package via the manifest.)
- **Repo has heavy customization** (custom labels, custom milestone naming). Honor; reflect in the generated file rather than forcing conventions.

## Distinguishing from other skills

- **vs. `documentation-check`** — that runs doc hygiene against existing files; this creates the metadata file that those skills then read.
- **vs. `repo-audit`** — audit reads `.repo-meta.yaml` to know the layout; this skill generates the file.

## What this skill never does

- Never commits. Writes to the working tree only.
- Never forces a layout. Honors what's detected.
- Never overwrites an existing `.repo-meta.yaml` without showing a diff and asking.
- Never creates GitHub labels or milestones. (That's a separate explicit action.)
