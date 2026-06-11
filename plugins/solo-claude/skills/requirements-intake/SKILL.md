---
name: requirements-intake
description: Turn a vague brief into a structured intake document for new packages, modules within an existing package, library extensions, or expansions to an existing package or app. Classifies the work type early, then runs up to three rounds of clarifying questions about problem, users, success metric, non-goals, integration shape, data, SLA, and constraints. Produces intake.md and intake.json under a per-brief folder in .claudedocs. Use when a brief is short or fuzzy. Trigger phrases include "plan a new feature", "scope this idea", "we want to build", "scope a new package", "expand the api package to handle X", "extract a shared library for Y", "what would it take to". First skill in the plan-new-work chain. Output feeds package-context-scan. Schema version 1.0.
---

# requirements-intake

Turns a vague brief into a structured intake. First phase of `plan-new-work`.

## What it produces

- `.claudedocs/<slug>/intake.md`
- `.claudedocs/<slug>/intake.json` — `schema_version: "1.0"`, consumed downstream.

`<slug>` is kebab-case from the brief, max 40 chars. Resume offered if the folder already exists.

## Trigger phrases

```
"Plan a new feature for ..."
"Scope this idea: ..."
"What would it take to ..."
"Scope a new package for ..."
"Add a CLI to the api package."
"Extract a shared library for ..."
```

## Phase 1 — Classify work type

Confirm one of four — don't guess silently:

1. **new-package** — a wholly new workspace package.
2. **new-module** — a logically distinct subsystem inside an existing package.
3. **library-extension** — adds or extends a shared library used by other packages.
4. **package-expansion** — adds capability to an existing package without a new module boundary.

For `single-package` repos, options 1 and 3 effectively collapse to "new top-level subsystem". Surface both choices but explain the collapse.

Capture as `work_type`.

## Phase 2 — Round 1 (always)

Single multi-part question:

1. **Problem** — what's the underlying pain, not what to build.
2. **Primary users** — internal team, end users, downstream package, operators.
3. **Success metric** — how do we know in 90 days this was the right call. Prefer measurable.
4. **Non-goals** — what's explicitly out of scope.

If the brief already states any of these, summarize and confirm rather than re-ask.

## Phase 3 — Round 2: work-type-specific

### new-package
- What capability lives here that doesn't belong in any existing package?
- Data shape: does this package own new state or surface existing state?
- Interaction with other packages — direct import, event bus, HTTP boundary, CLI?
- Performance expectations (latency, throughput, memory footprint)?
- Who maintains it long-term?

### new-module
- Which existing package is the host?
- Why a module rather than a new package or a library extension?
- New external dependencies (DB tables, file paths, vendor APIs)?
- Will the module have its own test runner / build step, or ride the host's?

### library-extension
- Which library? Which packages consume it today?
- Is this additive (new functions) or a behavior change?
- Versioning policy — semver expectation?
- Migration story for current consumers — coordinated, gradual, no-op?

### package-expansion
- Which package?
- Which existing endpoints / exports might be affected?
- New public API, new internal capability, or behavior change to existing?
- Feature-flag rollout vs. straight ship?

## Phase 4 — Round 3 (only if needed): constraints

Skip if the prior rounds didn't surface constraints.

- Hard deadlines?
- Must reuse / must avoid specific tech?
- Budget for new infra (new DB, new vendor)?
- Compliance / data-residency?
- Solo time-availability constraint (no, really — solo devs need to be honest about this)?

## Phase 5 — Open questions

Capture as `open_questions` in JSON. Surface back to the user at the end.

## Phase 6 — Write artifacts

### intake.md

```
# Intake: <one-liner>

**Work type:** ...
**Slug:** ...
**Captured:** <ISO date>
**Source:** verbal | issue-NNN | file-attachment | etc.

## Problem
## Users
## Success metric
## Non-goals
## Classification-specific answers
## Constraints
## Open questions
## Raw brief (verbatim)
```

### intake.json

```json
{
  "schema_version": "1.0",
  "slug": "...",
  "captured_at": "2026-06-11",
  "work_type": "new-package | new-module | library-extension | package-expansion",
  "summary": "...",
  "problem": "...",
  "users": ["..."],
  "success_metric": {"description": "...", "target": "...", "horizon": "..."},
  "non_goals": ["..."],
  "classification_details": { /* branch-specific */ },
  "constraints": {
    "hard_deadlines": [],
    "must_reuse": [],
    "must_avoid": [],
    "budget_notes": "",
    "compliance": []
  },
  "open_questions": [
    {"id": "Q-001", "question": "...", "blocking": true}
  ],
  "raw_brief": "..."
}
```

## Phase 7 — Hand off

Tell the user: "Intake captured. Next: `package-context-scan` to map intra-monorepo impact, or run `plan-new-work` to chain through to delivery plan."

## Edge cases and rules

- **Max 3 clarification rounds.** Past that, ship the intake with thin answers captured in `open_questions`.
- **Brief is a GitHub issue.** Pull via GitHub MCP; use issue number as part of slug; capture URL in `source`.
- **Resume.** Existing `intake.json` → ask resume or restart.
- **Brief is too vague to classify.** Force one targeted question; default to highest-cost-of-being-wrong (`new-package`) if still unclear.
- **Sensitive content.** Redact customer identifiers in `raw_brief` before writing.
- **Single-package repo + work_type `library-extension`.** Reclassify as `new-module` after confirming with user.

## Distinguishing from other skills

- **vs. `functional-requirements`** — that skill produces developer-ready requirements from an already-scoped brief. This shapes the vague brief into a structured intake first.
- **vs. `github-issue-creator`** — that creates GitHub issues from a scoped description. This skill produces the structured plan that downstream skills hand to issue-creator.

## What this skill never does

- Never invents requirements.
- Never proposes a solution. That's `solution-design`'s job.
- Never writes outside `.claudedocs/<slug>/`.
- Never commits.
