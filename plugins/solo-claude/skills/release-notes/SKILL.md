---
name: release-notes
description: Generate release notes from git history plus closed GitHub milestones and issues. Walks commits between two refs (or since the last tag), groups by Conventional Commit type, enriches with issue titles and labels via the GitHub MCP, classifies into Breaking / Features / Fixes / Internal / Other, and produces output suitable for a GitHub release body, a CHANGELOG.md entry, or a plain-text summary. Reads .repo-meta.yaml for tag format and milestone pattern when present. Use whenever cutting a release, writing a changelog entry, or summarizing what shipped between two points. Trigger phrases include "write release notes", "generate the changelog", "what shipped since v1.2.0", "prep release notes", "summarize the milestone".
---

# release-notes

Produces release notes from git history plus closed milestones / issues. Default output is a GitHub-release-ready Markdown block.

## What it produces

In chat by default. Optional outputs:

- A `CHANGELOG.md` entry appended (with diff preview before write).
- A `.claudedocs/releases/<version>.md` draft.
- A short MS Teams / Slack-style variant for posting in chat.

## Phase 1 — Determine the range

In order of preference:

1. User-supplied range (`since v1.3.0`, `v1.3.0..v1.4.0`, `last 20 commits`).
2. `<latest tag>..HEAD` if a tag exists in the repo matching `.repo-meta.yaml`'s `release.tag_format` (default `v{version}`).
3. All commits on the current branch since the default branch — for pre-release preview.

If no tag exists at all, recommend creating a baseline tag and proceed with all-commits.

## Phase 2 — Walk commits and classify

For each commit in the range:

- Parse Conventional Commit prefix (`feat`, `fix`, `chore`, `docs`, `refactor`, `test`, `perf`, `style`, `revert`, `build`, `ci`).
- Detect breaking changes: `!` in the prefix (`feat!: ...`) or `BREAKING CHANGE:` in the body.
- Extract issue references from the subject and body (`#123`, `Closes #123`, `Fixes #123`).
- For each referenced issue, pull title and labels via the GitHub MCP. Cache to avoid duplicate fetches.

Classify each commit into one of:

- **Breaking** — any commit with `!` or `BREAKING CHANGE:`.
- **Features** — `feat` commits not flagged breaking.
- **Fixes** — `fix` commits.
- **Internal** — `chore`, `refactor`, `test`, `build`, `ci`, `style`, `perf` without user-visible impact.
- **Other** — anything that didn't parse as Conventional Commit. Flag at the end of the run.

## Phase 3 — Cross-reference closed milestones

If a milestone closed within the range matches `.repo-meta.yaml`'s `release.milestone_pattern`, pull its closed issues. Cross-check:

- Issues in the milestone that have no corresponding commit in the range → flag as "expected but missing".
- Commits referencing issues outside the milestone → flag as "shipped but not in milestone".

Both are surfaced at the end of the output for the user to reconcile, not corrected automatically.

## Phase 4 — Format

Default Markdown block for a GitHub release body:

```
## What's changed

### Breaking changes
- ... (#123)

### Features
- ... (#124)

### Fixes
- ... (#125)

### Internal
- ... (#126)

### Reconciliation
- Issues in milestone v1.4.0 with no commit found: #127
- Commits referencing issues outside the milestone: a1b2c3d closes #999
```

If the diff has only `Internal` items, format the section header as "Maintenance" instead — better signal that there's nothing user-facing.

If the user asks for an executive / stakeholder audience, drop `Internal` and `Other`, collapse them into a one-line "plus N internal changes" footer.

If the user asks for an MS Teams / Slack version, format as a short bullet list with the headline first, no per-section H3s.

## Phase 5 — Optional CHANGELOG write

If the user requests "append to CHANGELOG", read `.repo-meta.yaml`'s `release.changelog_path` (default `CHANGELOG.md`). Locate the unreleased section (or the appropriate version section if one exists). Show a diff. Write only after explicit confirmation.

## Edge cases and rules

- **No commits in range.** Tell the user; offer to switch the range or create a tag.
- **No Conventional Commits at all.** Group everything as "Other"; recommend adopting Conventional Commits going forward. Don't refuse to generate notes.
- **Squashed merges where the commit subject is the PR title.** Treat the PR title as the source of truth. The body is sometimes the merged commits' originals; sample them for context but don't render them all.
- **GitHub MCP not authenticated.** Generate notes from commits only; flag missing issue context with a one-line note at the top.
- **Pre-release tags** (`v1.4.0-rc.1`). Use them as range endpoints but don't include them as "current" if a final tag exists.
- **Merge commits.** Skip from the rendered output unless `--include-merges` requested. Their content is duplicated in the squashed children.

## Distinguishing from other skills

- **vs. `documentation-check`** — that skill checks the existence and freshness of CHANGELOG; this skill generates the entries.
- **vs. `workflow-hygiene-check`** — that skill flags hygiene problems in open issues; this skill summarizes closed work.

## What this skill never does

- Never tags a commit. Tag creation is a deliberate user action.
- Never publishes a GitHub release. The user pastes the output into the release page (or does it via the GitHub MCP separately).
- Never overwrites an existing CHANGELOG section without showing a diff and asking.
