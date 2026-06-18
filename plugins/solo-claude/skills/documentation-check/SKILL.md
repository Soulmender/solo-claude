---
name: documentation-check
description: Lightweight doc-hygiene scan across README.md, CHANGELOG.md, docs/, ADRs, and per-package READMEs in a monorepo. Checks presence, freshness vs. recent code activity, truthfulness of documented commands, link integrity, and coverage of public APIs. Produces a green / yellow / red checklist. Also maintains an optional freshness ledger that records the commit each doc was last verified against, so drift is detected continuously — when code under a doc's purview changes, the doc is flagged automatically rather than only on a manual run. Honors .repo-meta.yaml's docs.layout setting (docs-folder-plus-readme, mkdocs, docusaurus, readme-only). Use before cutting a release, when auditing a project's doc health, to find stale sections, or to wire a drift check into a hook or schedule. Trigger phrases include "check the docs", "is this repo's documentation up to date", "find stale docs", "run a doc health check", "audit the README and CHANGELOG", "what docs has my change made stale".
---

# documentation-check

A focused doc-hygiene pass. Lightweight enough to run before every release without burning context.

## What it produces

In-chat checklist with a green / yellow / red headline. Optional save to `.claudedocs/doc-checks/<date>.md`.

## Phase 1 — Determine scope

Read `.repo-meta.yaml`'s `docs` block to know what to check:

- `docs-folder-plus-readme` — scan `README.md`, `CHANGELOG.md`, `docs/` recursively, each package's `README.md` under `monorepo.packages_root`.
- `mkdocs` — same, plus `mkdocs.yml` (validates nav references existing files).
- `docusaurus` — same, plus `sidebars.*` and `docusaurus.config.*`.
- `readme-only` — `README.md` and per-package `README.md` only. No `docs/` checks.

If no `.repo-meta.yaml`, infer from filesystem and proceed; offer to scaffold one at the end.

## Phase 2 — Per-document checks

For each doc, evaluate:

- **Presence** — does this expected doc exist? Missing `README.md` is red. Missing per-package `README.md` is yellow per package.
- **Structure** — are expected sections present? README should have at least: title, one-paragraph description, install / run instructions. CHANGELOG should follow Keep-a-Changelog or similar.
- **Freshness** — when was the doc last touched (`git log -1`)? When was the relevant code last touched? If code touched within the last 60 days and doc not touched in the last 180, flag as stale.
- **Truthfulness of commands** — `npm run dev`, `pnpm test`, etc. Cross-check against `package.json` `scripts`. Flag commands that don't exist.
- **Link integrity** — internal links resolve to existing files / anchors. External links: don't fetch (too slow + flaky); flag only obviously-broken ones (typos in TLD, `http://localhost/...`).
- **Public-API coverage** — for libraries, the README should list exported functions / classes. Spot-check: pick 3 exports, search the README; if none mentioned, flag.

## Phase 3 — Cross-cutting checks

- **ADR drift** — for each `docs/adrs/NNNN-*.md`, check `Status`. ADRs marked `Proposed` for more than 90 days flagged.
- **CHANGELOG drift** — if commits exist after the last CHANGELOG entry's `## [version]`, flag.
- **Per-package README divergence** — if multiple packages have READMEs and they share boilerplate sections (install, contribute), check that the boilerplate is in sync; divergent copies = pick-one risk.
- **MkDocs / Docusaurus nav consistency** — files exist for every nav entry; nav entries exist for every file under `docs_root`.

## Phase 4 — Headline

Compute a single rollup:

- **Red** — README missing, install commands broken, or CHANGELOG/code drift greater than two minor versions.
- **Yellow** — stale sections, missing per-package READMEs, ADR drift, link rot.
- **Green** — everything else.

Present in this order in chat:

```
## Doc check: <date>
**Headline:** 🔴 / 🟡 / 🟢 — one-line summary

### Red
- ...
### Yellow
- ...
### Green
- ...
```

## Phase 5 — Freshness ledger (continuous drift signal)

The scan above is a point-in-time check. To keep a *growing* codebase honest between runs, this skill maintains a small **freshness ledger** that turns staleness from invisible into tracked.

The ledger lives at `.claudedocs/doc-checks/freshness-ledger.json` (`schema_version: "1.0"`) and maps each tracked doc (or doc section) to:

- `last_verified_commit` — the commit at which a human/agent last confirmed the doc matched the code.
- `watches` — the code paths whose changes should invalidate this doc (e.g. `docs/architecture.md` watches `packages/*/src`; a per-package README watches that package; a flow doc watches its participating files; `docs/code-map.json` watches all source).
- `generated_by` — which skill owns the doc (`docs-author`, `flow-docs`, `code-map`, etc.), so a refresh routes correctly.

Two modes:

- **Update mode.** After a doc is (re)generated or confirmed accurate, stamp `last_verified_commit = HEAD` for it. Other skills (`docs-author`, `flow-docs`, `code-map`, `domain-glossary`) call this on promotion so the ledger stays current without manual upkeep.
- **Drift mode.** Given a commit range (default: `last_verified_commit..HEAD` per doc, or a range the user/hook passes), diff the watched paths. Any doc whose watches changed since its `last_verified_commit` is reported as **drifted**, newest-divergence first. This is what a pre-commit/pre-push hook or a scheduled task invokes to answer "what did my change make stale" — output is a terse drift list, not the full hygiene scan.

The ledger is a signal, never an action: drift mode reports and optionally hands off to the owning skill; it never edits docs itself. Seed the ledger from the current scan if none exists; offer to add `.claudedocs/` to `.gitignore` if missing (same convention as the rest of the chain).

## Edge cases and rules

- **No ledger yet.** Drift mode seeds one from the current state (`last_verified_commit = HEAD` for every present doc) and reports "baseline established, no drift". Subsequent runs are meaningful.
- **A watched path was deleted/moved.** Flag the doc for review (its subject may be gone) rather than silently passing.
- **Doc edited but code unchanged.** Not drift; stamp it verified. Drift is strictly code-moved-since-doc-verified.
- **Hook/scheduled invocation.** Keep output minimal (the drift list + headline); skip the full Phase 1–4 scan unless asked. Never block a commit — report and let the user decide.
- **Repo has no `README.md` at all.** Red headline, single finding. Offer to bootstrap one via `docs-author`.
- **Multiple `CHANGELOG.md` files** (per-package + root). Check all; flag divergence if one is fresh and others stale.
- **README is auto-generated** (e.g. from JSDoc/TypeDoc). If a banner says so, skip freshness checks; still check command truthfulness.
- **Lockfile-only commits** between doc updates. Don't count as "code touched" for staleness purposes.
- **No `.repo-meta.yaml`.** Don't fail; warn at the bottom and proceed with filesystem inference.

## Distinguishing from other skills

- **vs. `repo-audit`** — this is the doc-hygiene 5-minute version. The audit is the architectural and code deep-dive that takes much longer.
- **vs. `docs-author`** — this skill diagnoses; `docs-author` generates / repairs. Run this before deciding whether docs-author is worth invoking.

## What this skill never does

- Never modifies docs. Reports only. Repairs are docs-author's job. The only file it writes is its own freshness ledger under `.claudedocs/`.
- Never blocks a commit. Drift mode reports; the user decides.
- Never fetches external links. Static checks only.
- Never grades code quality, test coverage, or anything outside documentation.
