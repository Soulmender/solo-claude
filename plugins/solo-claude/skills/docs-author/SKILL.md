---
name: docs-author
description: Generate or update living documentation from the audit JSON (and optionally verification JSON). Produces README.md (voice-preserving), docs/architecture.md, docs/runbook.md, per-package READMEs, and draft ADR stubs under docs/adrs. Honors .repo-meta.yaml's docs.layout (docs-folder-plus-readme, mkdocs, docusaurus, readme-only) and updates nav config when applicable. Preserves user-written sections via the same conventions as docs-update. Diff before write. Use after repo-audit when documentation needs to be (re)generated from audit findings. Trigger phrases include "update the docs from the audit", "write the README from the audit JSON", "regenerate the docs", "draft ADR stubs from the audit's architectural findings", "write up the runbook".
---

# docs-author

Turns an audit into living documentation. Companion to `repo-audit` and `repo-verifier`.

## What it produces

Depending on user choice (asked in Phase 1):

- `README.md` at the repo root — updated sections, voice preserved.
- `docs/architecture.md` — high-level architecture and the cross-package map.
- `docs/runbook.md` — operational runbook (how to run locally, deploy, debug).
- Per-package `packages/<pkg>/README.md` — per-package summary.
- `docs/adrs/NNNN-*.md` — ADR stubs for architectural findings flagged in the audit.
- MkDocs / Docusaurus nav updates when `docs.layout` indicates one of those.

Drafts land in `.claudedocs/docs/` first; promotion is gated.

## Phase 1 — Choose artifacts

Read `audit.json` (required). Read `verification.json` if available — it enriches the "recently shipped" and "false-Done" sections.

Ask the user which artifacts to generate:

```
1. README.md
2. docs/architecture.md
3. docs/runbook.md
4. per-package READMEs
5. ADR stubs from architectural findings
6. All of the above
```

For `docs.layout: readme-only`, options 2 and 3 are unavailable; their content folds into README sections.

For `docs.layout: mkdocs` or `docusaurus`, after generating a new file the skill also drafts the nav-config update.

## Phase 2 — Read current state

For each chosen artifact, read the existing file (if any):

- Identify preserved regions (`## Manual notes`, `<!-- preserve-start -->` blocks, anything listed in `.repo-meta.yaml`'s `docs.preserved_sections`).
- Identify "voice" sections — handcrafted intros, badges, jokes, contributor lists, sponsorship boxes. These get preserved verbatim.
- Identify "factual" sections — install instructions, configuration tables, API surface listings, architecture descriptions. These are eligible for regeneration.

## Phase 3 — Per-artifact generation

### README.md

Standard sections, all in factual-zone unless they have a voice intro:

- **Title** — current.
- **One-paragraph description** — from audit's executive summary, voice-matched.
- **Install** — from package manager detected in monorepo layout.
- **Run / develop** — actual commands from `package.json` `scripts` or `pyproject.toml` `[tool.uv.scripts]`.
- **Configuration** — env vars surfaced in audit; truthful only.
- **API surface (libraries only)** — list of exports from audit, with one-line summaries.
- **Project structure** — package map for monorepos, top-level folders for single-package.
- **Contributing** — link to `CONTRIBUTING.md` if exists; otherwise short defaults.
- **License** — from `LICENSE` file presence.

Sections under preserved-region headings are not touched. New factual content replaces old factual content with a diff.

### docs/architecture.md

- **High-level view** — Mermaid diagram of packages and major external systems.
- **Per-package summary** — role, primary responsibility, key dependencies.
- **Cross-package boundaries** — what each package exposes, what it consumes internally.
- **External integrations** — APIs, databases, queues, third-party services.
- **Significant patterns** — caching, queueing, state management, auth.
- **Open architectural questions** — pulled from audit's open-questions section.

### docs/runbook.md

- **Local development** — exact commands.
- **Test execution** — how to run tests for the whole monorepo and per package.
- **Build** — production build command per package.
- **Deploy** — only if audit's deploy info is reliable (CI workflows, deploy scripts). Otherwise skip with a note.
- **Common failure modes** — pulled from audit's "known risks" section.
- **Where to look first when something breaks** — log paths, observability links, health endpoints.

### Per-package READMEs

For each package, generate or refresh a minimal README:

- Name + one-line role.
- Install + usage (for libraries) or run command (for apps).
- Public API summary.
- Internal dependencies (which monorepo packages it consumes).

If multiple packages share a boilerplate section, write once and reference from each. Don't fan out duplicate text that's destined to drift.

### ADR stubs

For each `Major` architectural finding in the audit (e.g. cross-package cycle, new dependency, schema-shaping pattern), draft `docs/adrs/NNNN-<slug>.md` with `Status: Proposed`. Number sequentially.

```
# NNNN — <title>

**Status:** Proposed
**Date:** <today>

## Context
<from audit's finding context>

## Decision
<empty — user fills in>

## Consequences
<empty — user fills in>
```

User reviews, fills the decision, flips status to Accepted.

## Phase 4 — Diff and write

For each artifact:

1. Show a unified diff against the existing file (or full content if new).
2. Highlight preserved regions inline so the user sees they're untouched.
3. Wait for confirmation.
4. Write to `.claudedocs/docs/<artifact>` (draft).
5. Offer to promote to the final location.

Batch-confirm or per-artifact-confirm, user's choice.

## Phase 5 — MkDocs / Docusaurus nav updates

If `docs.layout` is `mkdocs` or `docusaurus` and the artifact is new (not previously listed in nav), generate the nav-config edit too. Show diff; write only with confirmation.

## Edge cases and rules

- **Stale audit (greater than 1 month old, or before substantial code changes).** Warn at the start; let the user proceed or re-audit first.
- **No audit JSON.** Exit with: "Run `repo-audit` first."
- **Existing README has heavy voice content.** Preserve aggressively; regenerate only sections clearly marked as factual (Install, Run, etc.) or whose headings match well-known patterns.
- **A package has no public API.** Skip the API section; don't write an empty one.
- **MkDocs nav has a hand-curated order.** Insert new entries alphabetically within the existing top-level group, not at the top. Show diff loudly.
- **Auto-generated README banner present.** Refuse to edit; recommend updating the generator source.
- **Sensitive content** in audit (customer names, infra internals) — redact before writing to public-facing docs per team CLAUDE.md.

## Distinguishing from other skills

- **vs. `docs-update`** — that skill edits one specific file based on a verbal description. This skill regenerates structured artifacts from an audit. Different inputs, different cadence.
- **vs. `documentation-check`** — that skill diagnoses doc hygiene; this skill repairs.

## What this skill never does

- Never overwrites preserved regions.
- Never auto-promotes drafts.
- Never auto-commits or auto-pushes.
- Never fabricates content not supported by the audit. If audit didn't find it, this skill doesn't write it.
