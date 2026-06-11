---
name: docs-update
description: Edit one specific Markdown file in the repo based on a verbal description. Reads the current file, drafts the change, asks where to insert if ambiguous, preserves the user's manual-notes sections and HTML-comment preservation blocks, shows a diff, writes only after confirmation. Targets README.md, files under docs/, ADRs, RFCs, per-package READMEs, CONTRIBUTING.md — any in-repo Markdown. Use for one-off edits — fix a typo, add a section, update a status block, append a paragraph. Trigger phrases include "update the README to mention X", "edit the architecture doc to add the new caching layer", "fix the typo on page X", "add a deprecation note to docs/api.md".
---

# docs-update

Ad-hoc edits to a single Markdown file in the repo. The lightweight counterpart to `docs-author` (which regenerates structured artifacts from an audit).

## What it produces

A modified Markdown file in the repo. Diff shown before write; written only on confirmation.

## Phase 1 — Identify the file

The user names the file by path, by topic, or by role:

- **Path** (`README.md`, `docs/architecture.md`) — read directly.
- **Topic** ("the architecture doc") — search `.repo-meta.yaml`'s `docs_root` for a matching filename or H1, confirm with user.
- **Role** ("the runbook") — look for `docs/runbook.md`, `docs/operations.md`, or similar; confirm.

If multiple candidates, list them and ask. Don't guess.

## Phase 2 — Understand the change

Parse the user's request into one of:

- **Add a section** — "add a section about X". Asks where (top, before `<named existing section>`, after `<named>`, append).
- **Edit a section** — "update the 'Status' section to reflect Y". Identifies the section by heading; confirms if multiple matches.
- **Fix a typo / factual error** — "find 'Postgress' and change to 'Postgres'". Confirms which instances if there are multiple matches.
- **Append a note / paragraph** — "add a deprecation note about Z at the top". Asks for placement.
- **Restructure** — "move section A above section B". Confirms before reordering.

## Phase 3 — Draft the change

Generate the proposed new content. Match the file's existing voice:

- If the doc is terse, write terse.
- If it uses bullets, use bullets.
- If it has examples, include one for the new content.

For technical content, preserve any consistent format the file already uses (code-block language tags, heading levels, table layouts).

## Phase 4 — Preservation rules

The skill never overwrites:

1. Sections under `## Manual notes` (or any heading listed in `.repo-meta.yaml`'s `docs.preserved_sections`).
2. Anything between `<!-- preserve-start -->` and `<!-- preserve-end -->` HTML-comment markers.
3. Front-matter blocks in MkDocs / Docusaurus files (the YAML at the top between `---` lines).
4. The `<!-- generated -->` banner blocks at the bottom of auto-generated files; refuse to edit those files and recommend updating the source instead.

If the requested edit would touch a preserved region, refuse with a clear "this section is preserved — edit it manually" message.

## Phase 5 — Diff + confirm + write

Show a unified diff. Wait for user confirmation. Write.

After writing, suggest the commit message:

```
docs(<area>): <one-line summary of the change>
```

Where `<area>` is the doc's location (`readme`, `docs/architecture`, `docs/adrs/0007`).

If the change should be paired with a code change (e.g. updating README install instructions while changing `package.json` scripts), surface that to the user before writing.

## Phase 6 — Optional cross-doc updates

If the edit touches a referenced section in another doc — say, you updated `docs/api.md` and the README links to it — surface the cross-reference and ask whether to update the README's link text too. Don't auto-cascade.

## Edge cases and rules

- **The file doesn't exist.** Offer to create it; clarify shape before doing so. Don't silently create.
- **The section the user named doesn't exist.** Tell them; offer to create it (and ask where).
- **Multiple sections share the same heading.** List them with their line numbers; ask which.
- **The edit would change a link target referenced elsewhere.** Warn; ask whether to update referrers.
- **Code blocks need updating** alongside prose (e.g. a "run this command" example whose command changed). The skill updates them too; flags the pair for the user to confirm.
- **MkDocs `nav` or Docusaurus `sidebars` references** the file being edited under a stale name. Flag, but don't reorganize navigation here (that's `docs-author`'s job).

## Distinguishing from other skills

- **vs. `docs-author`** — that skill regenerates structured artifacts from an audit. This skill edits one specific file based on a verbal description. Use docs-author after audits; use docs-update for ad-hoc fixes.
- **vs. `documentation-check`** — that skill diagnoses; this one repairs a specific finding.

## What this skill never does

- Never edits more than one file per invocation (except the explicit cross-doc update step, which is gated by user confirmation).
- Never modifies preserved regions, no matter what the user asks.
- Never commits or pushes. Writes to the working tree only.
- Never auto-creates a PR.
