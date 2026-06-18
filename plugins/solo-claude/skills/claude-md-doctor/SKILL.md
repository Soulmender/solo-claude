---
name: claude-md-doctor
description: Audit and improve the CLAUDE.md files in a repo so agents actually follow the conventions the codebase already established. Discovers every CLAUDE.md (repo root, per-package, and the symlinked global solo-CLAUDE.md), cross-checks what they instruct against the patterns the code really uses (from patterns.json or audit.json) and against the team conventions, and flags gaps, contradictions, stale rules, and vague guidance. Proposes concrete diff-gated edits — add the missing convention, fix the contradiction, scope a rule to the right package — preserving hand-written sections. Honors .repo-meta.yaml. Never auto-commits. Use to make agent guidance match reality, before onboarding a new package, or after a convention changes. Trigger phrases include "improve our CLAUDE.md", "do the CLAUDE.md files match how we actually code", "fix agent instructions", "audit CLAUDE.md", "the agents aren't following our conventions", "add the new convention to CLAUDE.md".
---

# claude-md-doctor

Keeps CLAUDE.md honest. A CLAUDE.md is only useful if it tells agents to do what the repo actually does — this skill finds where guidance and reality drift apart and repairs the guidance.

## What it produces

- An in-chat **CLAUDE.md health report** with a green / yellow / red headline, per-file findings, and a proposed-edits list.
- Diff-gated edits to each CLAUDE.md (root, per-package, and the global template when in scope). Drafts written to `.claudedocs/claude-md/` first; promotion to the real file is user-confirmed.
- Optional `.claudedocs/claude-md/report.md` saved copy of the report.

This skill *writes* (it's a repair skill, like `docs-author` / `docs-update`), but only to CLAUDE.md files, only after a diff, and never auto-commits.

## Inputs

- Every `CLAUDE.md` in the repo: root, any `packages/*/CLAUDE.md`, and the global `~/.claude/CLAUDE.md` (the symlinked `solo-CLAUDE.md` template) when the user opts to include it.
- `patterns.json` from `design-patterns` if present — the primary source of truth for "what the conventions actually are". Strongly recommended; offer to run `design-patterns` first if missing.
- `audit.json` from `repo-audit` if present — supplements with security findings, dependency facts, and the package map.
- `.repo-meta.yaml` for layout, package roles, and the security/"never" conventions.

## Phase 1 — Discover and classify CLAUDE.md files

Enumerate all CLAUDE.md files. For each, classify regions:

- **Preserved / hand-written** — narrative, rationale, "why" notes, anything under `## Manual notes`, `## Maintainer notes`, or `.repo-meta.yaml` `docs.preserved_sections`, plus obvious voice content. Never rewritten.
- **Convention rules** — imperative instructions ("Use X", "Never Y", "Always Z"). Eligible for audit and repair.
- **Context / orientation** — descriptions of the repo, package roles, where things live. Eligible for freshness fixes.

Establish the layering: global template (cross-repo rules) → root CLAUDE.md (this repo) → package CLAUDE.md (this package). A rule belongs at the *most specific* level that fully owns it.

## Phase 2 — Cross-check guidance against reality

For each convention rule, evaluate against `patterns.json` / `audit.json` / `.repo-meta.yaml`:

- **Gap** — a `consistent-undocumented` or `contested` pattern the code follows that no CLAUDE.md mentions. (The highest-value finding — these are why agents "don't follow conventions": nobody told them.)
- **Contradiction** — a rule that tells agents to do the opposite of what the code does (e.g. CLAUDE.md says "throw exceptions" but the house style returns `Result` types). Either the rule is stale or the code is the exception — surface it; let the human decide which to change.
- **Stale** — a rule referencing a tool, path, package, or script that no longer exists (cross-check `package.json` scripts, folder layout, dependency list).
- **Vague** — a rule too soft to act on ("write good tests", "follow best practices"). Propose a concrete version grounded in the actual testing/style pattern.
- **Misplaced** — a package-specific rule sitting in the root file, or a repo-wide rule buried in one package. Propose relocation.
- **Redundant / conflicting across layers** — the same rule duplicated, or a package file contradicting the root. Propose dedupe or explicit override.

Also check **coverage**: for each `candidates_for_claude_md` dimension in `patterns.json`, is there a corresponding rule somewhere in the CLAUDE.md layering? Missing ones become proposed additions.

## Phase 3 — Structural and hygiene checks

- **Security "never" rules present?** Compare against the global template's security block (no secrets in files, parameterized SQL, no `eval` on input, no real customer data, no direct push to `main`). A repo missing these in any form is a yellow finding.
- **Per-package CLAUDE.md needed?** If a package has a clearly distinct stack or convention (per `patterns.json` divergences), suggest a package-level CLAUDE.md scoped to it.
- **Length and focus.** Flag a CLAUDE.md that's so long agents will ignore the tail, or padded with generic advice that isn't repo-specific. Recommend trims.
- **Actionability.** Rules should be imperative and testable. Flag declaratives that should be instructions.

## Phase 4 — Headline

- **Red** — a contradiction that will actively make agents write wrong code, or missing security "never" rules.
- **Yellow** — gaps for `contested`/`consistent-undocumented` patterns, stale references, vague rules, misplacement.
- **Green** — guidance matches reality; only nits.

## Phase 5 — Propose and apply edits

For each finding with a fix, draft the precise edit:

- **Additions** — new rule text, written in the file's existing voice and the imperative style of the surrounding rules, placed at the correct layer and section. Each new rule cites the evidence ("returns `Result` — see packages/api/src/errors.ts:12") in the report, not in the file.
- **Fixes** — minimal edit to the offending line; show old → new.
- **Relocations** — remove from one file, add to another; both diffs shown together.

Then:

1. Present all proposed edits grouped by file, each as a unified diff, preserved regions highlighted as untouched.
2. Let the user accept all, accept per-file, or cherry-pick per-edit.
3. Write accepted edits to `.claudedocs/claude-md/<path>` drafts.
4. Offer to promote drafts to the real CLAUDE.md locations.
5. For a **contradiction**, never silently rewrite — present both directions ("change the rule to match the code" vs. "the code is the outlier") and let the user choose; offer to record the resolution via `decision-ledger`.

## Edge cases and rules

- **No CLAUDE.md anywhere.** Offer to bootstrap a root CLAUDE.md from `patterns.json` + the global template, rather than failing. Bootstrapped file is draft-first, diff-gated like everything else.
- **No patterns.json and no audit.json.** Warn that cross-checks against reality will be shallow; offer to run `design-patterns` first. Can still do structural/staleness/security checks without them.
- **Global template (`~/.claude/CLAUDE.md`) is a symlink to the marketplace.** Editing it changes the shared template for every repo. Require explicit opt-in to touch it; default scope is repo-local CLAUDE.md files only. Warn loudly about the blast radius.
- **Contradiction where the code is wrong, not the rule.** This skill does not change code. Note that the fix may belong in the code (hand off to `improvement-advisor` / an issue) and leave the rule intact unless the user says otherwise.
- **Voice-heavy CLAUDE.md.** Preserve aggressively; touch only clearly-imperative rule lines.
- **Monorepo with 20+ packages.** Don't propose 20 package CLAUDE.md files. Recommend package files only where divergence justifies them; keep shared rules at root.
- **Sensitive content.** Never write secrets, customer names, or infra internals into a CLAUDE.md, even if found in the code. Redact in the report per the team "never" rules.

## Distinguishing from other skills

- **vs. `design-patterns`** — that skill discovers what the conventions are. This skill checks whether the CLAUDE.md files tell agents to follow them, and fixes the guidance. design-patterns is read-only; this one repairs CLAUDE.md.
- **vs. `docs-update`** — docs-update edits one arbitrary Markdown file from a verbal description. This skill is purpose-built for CLAUDE.md: it knows the layering, the security baseline, and how to cross-check against `patterns.json`.
- **vs. `documentation-check`** — that skill checks README/CHANGELOG/docs hygiene for humans. This skill checks agent-facing guidance against code reality.
- **vs. `scaffold-repo-meta`** — that writes `.repo-meta.yaml` (machine context). This writes CLAUDE.md (agent instructions). Complementary.

## What this skill never does

- Never edits anything but CLAUDE.md files (and only their convention/context regions).
- Never touches preserved / hand-written regions.
- Never edits code to resolve a contradiction — it surfaces the choice; code changes route to `improvement-advisor` or an issue.
- Never edits the global symlinked template without explicit opt-in.
- Never auto-promotes drafts or auto-commits.
- Never writes a rule it can't ground in evidence (a pattern, a `.repo-meta.yaml` field, or a security baseline). No generic filler advice.
