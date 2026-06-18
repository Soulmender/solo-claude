---
name: package-context-author
description: Generate scoped, agent-facing context files for each package in a monorepo so a coding agent gets local conventions right where it's working — not just a repo-wide CLAUDE.md. For each package, writes a concise per-package CLAUDE.md (or CONTEXT.md) covering the package's role, public surface, internal dependencies in and out, local conventions and gotchas, test commands, and explicit "do not do X here" rules. Sources from audit.json, patterns.json, and the package manifest; inherits the root CLAUDE.md and only states what differs locally. Diff-gated, preservation-aware, draft-first; never auto-commits. Use to give agents local context in a growing monorepo, when onboarding a new package, or after a package's conventions diverge. Trigger phrases include "generate per-package context", "give each package its own CLAUDE.md", "write local agent context for packages/api", "cascade conventions into each package".
---

# package-context-author

Pushes context down to where code is written. A repo-wide `CLAUDE.md` tells an agent the house rules; this skill gives each package the local rules, surface, and gotchas an agent needs when editing *that* package — the highest-leverage context for code quality in a growing monorepo.

## What it produces

- `packages/<pkg>/CLAUDE.md` (or `CONTEXT.md` if the user prefers to reserve `CLAUDE.md` for hand-written rules) per selected package.
- Each file is intentionally short — a coding agent reads it on every edit, so it must be scannable, not a second architecture doc.

Drafts land in `.claudedocs/package-context/<pkg>/` first; promotion to the package folder is user-gated. Never auto-commits.

## Inputs

- `audit.json` (`repo-audit`) — per-package role, public surface, internal deps, known risks. Primary source.
- `patterns.json` (`design-patterns`) — house style plus this package's divergences (the local exceptions worth pinning).
- The package manifest (`package.json` / `pyproject.toml`) — scripts, exports, deps.
- Root `CLAUDE.md` — so the per-package file inherits rather than repeats. Only local *differences* are written.

If `audit.json` / `patterns.json` are absent, do a lighter direct scan of the package and lower confidence; recommend running `repo-audit` + `design-patterns` first.

## Phase 1 — Select packages

Read the package list from `.repo-meta.yaml` (or infer). Default offer: packages with an `app` or `library` role, or any package `design-patterns` flagged as divergent (those benefit most). Let the user widen or narrow. For a single-package repo, produce one root-adjacent `CONTEXT.md` scoped to the main module tree instead.

## Phase 2 — Compute the local delta against the root

For each package, derive only what an agent couldn't infer from the root `CLAUDE.md`:

- **Role and boundary** — what this package is for, and what it must NOT take on (boundary discipline).
- **Public surface** — the exports other packages depend on; "changing these is a breaking change."
- **Internal deps** — what it consumes from the monorepo, what consumes it (blast radius if changed).
- **Local conventions** — where this package's `patterns.json` entry differs from the house style, stated as an instruction ("errors here use X, unlike the repo default — see anchor").
- **Gotchas / known risks** — from the audit's findings for this package (without restating fixable bugs; that's `improvement-advisor`).
- **Commands** — the actual build/test/lint commands for this package, from its manifest scripts.
- **Do-not rules** — concrete local prohibitions (e.g. "do not import from `packages/web` here — this is a leaf library").

Anything identical to the root file is omitted with a one-line "inherits root CLAUDE.md."

## Phase 3 — Write each file

Target shape (kept under ~40 lines):

```
# <pkg> — agent context

Inherits the repo root CLAUDE.md. Below is what's specific to this package.

**Role:** <one line>. **Boundary:** <what it must not do>.

**Public surface (breaking if changed):** <exports>

**Depends on (internal):** <in>. **Depended on by:** <out — blast radius>.

**Local conventions**
- <dimension>: <instruction> (differs from house style — see <anchor>)

**Commands:** build `…` · test `…` · lint `…`

**Gotchas**
- <known risk an editor must know>

**Do not**
- <concrete local prohibition>
```

## Phase 4 — Diff, gate, promote

1. Show each file as a diff against any existing per-package context (full content if new), preserved/hand-written regions highlighted as untouched.
2. Accept all / per-package / per-section, user's choice.
3. Write accepted drafts to `.claudedocs/package-context/<pkg>/`.
4. Offer to promote to `packages/<pkg>/`.
5. Offer to add a one-line pointer in each package's existing README ("agent context: ./CLAUDE.md") and to register the files in the repo `context_index` (see `scaffold-repo-meta`).

## Edge cases and rules

- **Package already has a hand-written CLAUDE.md.** Treat all of it as preserved; only propose additive sections (surface, commands, blast radius) and flag contradictions for `claude-md-doctor` rather than overwriting.
- **Tiny / trivial package.** Skip — a 3-file utility doesn't need its own context file. Note the skip; don't pad.
- **No audit / patterns.** Direct-scan; mark confidence lower; keep the file shorter (surface + commands only).
- **Generated package.** Skip; its conventions belong to the generator.
- **Monorepo with many packages.** Don't fan out duplicate boilerplate — shared rules stay in the root; per-package files carry only deltas.
- **Sensitive content.** Never write secrets, customer names, or infra internals; redact per the team "never" rules.

## Distinguishing from other skills

- **vs. `claude-md-doctor`** — that skill audits and repairs existing CLAUDE.md against reality. This skill generates fresh per-package context where none exists. Hand off contradictions in existing files to `claude-md-doctor`.
- **vs. `docs-author` per-package READMEs** — those are human-facing (install/usage). This is agent-facing, instruction-grade, and delta-only against the root.
- **vs. `design-patterns`** — that catalogs conventions repo-wide and read-only. This turns the per-package divergences into local instructions.

## What this skill never does

- Never overwrites hand-written CLAUDE.md content.
- Never restates the root CLAUDE.md — local deltas only.
- Never lists fixable bugs as gotchas (that's `improvement-advisor`); only states risks an editor must respect.
- Never auto-promotes drafts or auto-commits.
- Never writes a rule without evidence (an audit finding, a pattern divergence, or a manifest fact).
