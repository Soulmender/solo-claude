---
name: plan-new-work
description: End-to-end orchestrator that turns a vague brief into a finished delivery plan plus optional downstream artifacts. Chains requirements-intake, package-context-scan, solution-design, and delivery-plan with explicit approval gates between phases, then optionally hands off to github-issue-creator for GitHub issue creation, drafts an ADR per ADR-worthy decision, drafts an in-repo RFC under docs/proposals, and chains into scaffold-repo-meta when the planned work creates a new package or the file is missing. Resumable — re-running on the same brief picks up from the last completed phase via .claudedocs artifacts. Pre-flights .gitignore for the .claudedocs convention. Use for any planning effort larger than a single issue. Trigger phrases include "plan this", "run the planning pipeline", "scope and plan", "take this from brief to plan", "end-to-end plan for". Wrapped by the /plan slash command.
---

# plan-new-work

End-to-end orchestrator. Chains the four planning skills with approval gates; offers downstream actions after Gate 4.

## What it produces

`.claudedocs/<slug>/` directory:

- `intake.{md,json}`
- `package-context.{md,json}`
- `design.{md,json}`
- `delivery-plan.{md,json}`
- `orchestrator-state.json` — phase progress, last completed gate, downstream actions taken.

Plus, on user approval, any of:

- GitHub parent issue + child issues created via `github-issue-creator`.
- ADR draft(s) at `docs/adrs/NNNN-<slug>.md`.
- RFC draft at `docs/proposals/<slug>.md` (or `.claudedocs/<slug>/rfc.md` if no `docs/proposals/`).
- `.repo-meta.yaml` scaffolded for a new package.
- Decisions recorded into `.decisions.yaml`.

## Phase 0 — Pre-flight, brief, and slug

### 0a — `.gitignore` pre-flight

Before writing artifacts, check `.claudedocs/` is gitignored:

1. Walk up to find `.git/`. None → user is running outside a repo; warn and write to `./.claudedocs/<slug>/` anyway.
2. Find the repo-root `.gitignore`.
3. Check for `.claudedocs/`, `.claudedocs`, or `**/.claudedocs/`. Use `git check-ignore .claudedocs/` if available.

Outcomes:

- Covered → continue silently.
- `.gitignore` exists, missing the entry → offer to append `.claudedocs/` with a one-line comment.
- No `.gitignore` → offer to create one with `.claudedocs/` and `.claude/`.
- No `.git` → warn explicitly; proceed.

Capture in `orchestrator-state.json.preflight.gitignore_status` so future resumes don't re-prompt.

### 0b — Brief and slug

Read the brief (verbal, file, GitHub issue number). Generate kebab-case slug. Confirm with user — it's the directory name; hard to change later.

Check whether `.claudedocs/<slug>/` exists:

- Doesn't exist → create + initialize state.
- Exists with `orchestrator-state.json` → show phase + last gate; ask resume or restart.
- Exists without state → likely manual sub-skill run; ask how to proceed.

## Phase 1 — requirements-intake

Invoke. After completion, verify `intake.json`. Update state.

### Gate 1
Summarize: work type, problem, primary users, success metric, open-questions count.
Ask: "Proceed to package-context-scan, edit the intake, or stop?"

## Phase 2 — package-context-scan

Invoke. After completion, verify `package-context.json`. Update state.

### Gate 2
Summarize: packages scanned, status counts, integration touchpoints, top gaps.
Ask: "Proceed to design, fill biggest gap first, or stop?"

If gaps look likely to invalidate the design, recommend filling them; don't hard-gate.

## Phase 3 — solution-design

Invoke. After completion, verify `design.json`.

### Gate 3
Summarize: recommendation, ADR count, top risk.
Ask: "Proceed to delivery plan, push back on the recommendation, or stop?"

If push back, re-run Phase 3 with user input.

## Phase 4 — delivery-plan

Invoke. After completion, verify `delivery-plan.json`.

### Gate 4
Summarize: phase counts, total children, biggest risk, top open questions.
Ask: "Proceed to downstream actions, push back on the breakdown, or stop?"

## Phase 5 — Optional downstream actions

Each is opt-in and gated.

### 5a — GitHub issue creation
Offer: "Hand the delivery plan to `github-issue-creator`? This will create one parent issue + child issues in <owner>/<repo>."
Warn: issues are created (not drafted) on confirmation; cleanup is manual.
If yes, invoke `github-issue-creator` with `delivery-plan.json` and the GitHub coordinates from `.repo-meta.yaml`.

### 5b — ADR drafts
If `design.json.adr_worthy_decisions` non-empty: offer to draft N ADRs.
If yes, draft each at `docs/adrs/NNNN-<slug-fragment>.md` with `Status: Proposed`. Number sequentially based on existing ADRs.

### 5c — In-repo RFC / proposal
Offer: "Draft an RFC / proposal page from the plan?"
If yes, write `docs/proposals/<slug>.md` (or `.claudedocs/<slug>/rfc.md` if no proposals folder). Honor preservation conventions.

### 5d — Scaffold a new package's metadata
If `intake.json.work_type == "new-package"`: offer to invoke `scaffold-repo-meta` (extending `.repo-meta.yaml` to register the new package).
If repo has no `.repo-meta.yaml` at all: offer to bootstrap it now.

### 5e — Record key decisions
After plan approval: offer to record the design recommendation + rollout choice to `.decisions.yaml` via `decision-ledger`.

## Phase 6 — Done

Tell the user:

- Where the artifacts are.
- What downstream actions ran successfully.
- Open questions still needing answers.
- Minimum next step.

Update state to `phase: "complete", completed_at: <now>`.

## Edge cases and rules

- **User stops at any gate.** State persisted; resume by re-running on the same brief or slug.
- **Sub-skill fails.** Stop chain at that phase; surface failure clearly; persist `orchestrator-state.json.phase: "<phase>-failed"`. Don't rollback successful artifacts.
- **Brief changes mid-flow.** Treat as restart — new slug, leave old folder.
- **Skip a phase** (user asks for direct delivery-plan). Honor with a clear warning; package-context-scan is the most dangerous to skip.
- **Schema mismatch between sub-skill outputs.** Refuse to chain; surface what to upgrade.
- **No `.repo-meta.yaml` for a package-expansion brief.** Offer `scaffold-repo-meta` first.
- **Multiple briefs in parallel.** Each gets own slug; orchestrator doesn't cross-correlate.
- **User declined gitignore offer in Phase 0a.** Honor; warn once; don't re-nag.
- **Sensitive content.** Per CLAUDE.md, redact customer identifiers before downstream actions write to GitHub or other shareable surfaces.

## Distinguishing from other skills

- **vs. each sub-skill standalone** — sub-skills are useful individually. Use the orchestrator for end-to-end runs with gates.
- **vs. `github-issue-creator`** — that's fine when work is already scoped and just needs issues. Use orchestrator when the brief is vague.
- **vs. `docs-author`** — that's post-hoc for an existing service. This is pre-hoc for work that doesn't exist yet.

## What this skill never does

- Never auto-confirms a gate.
- Never writes to GitHub / `docs/` / repo files without an explicit per-step confirmation.
- Never deletes prior artifacts. Restart creates a new slug; old slugs are kept.
- Never invents content the sub-skills didn't produce. The orchestrator chorographs; doesn't author.
