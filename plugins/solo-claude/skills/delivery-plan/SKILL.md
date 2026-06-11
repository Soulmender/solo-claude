---
name: delivery-plan
description: Turn an approved solution design into a phased, issue-ready delivery plan. Breaks work into P0/P1/P2 with explicit cuts. Builds a parent epic plus child issues honoring GitHub Issues conventions from .repo-meta.yaml (type and area labels, milestone). Sizes each item, identifies dependencies, lists risks with mitigations, and drafts rollout plus rollback strategy. Use after solution-design. Trigger phrases include "break it into issues", "draft the plan", "phase the work", "what does a delivery plan look like", "size this out". Reads intake.json, package-context.json, design.json. Writes delivery-plan.md and delivery-plan.json with schema version 1.0. Output is consumable by github-issue-creator for the actual issue-creation step.
---

# delivery-plan

Fourth phase of `plan-new-work`. Turns the recommended design into an issue-ready breakdown.

## What it produces

- `.claudedocs/<slug>/delivery-plan.md`
- `.claudedocs/<slug>/delivery-plan.json` — `schema_version: "1.0"`, consumed by `github-issue-creator`.

## Inputs

- `.claudedocs/<slug>/intake.json`
- `.claudedocs/<slug>/package-context.json`
- `.claudedocs/<slug>/design.json`

Exit if any missing.

## Phase 1 — Cut into P0 / P1 / P2

Apply ruthlessly:

- **P0 — must-have for the success metric to be testable.** If P0 ships and nothing else does, can you measure whether the recommendation works?
- **P1 — should-have.** Improves the experience; success metric already measurable without it.
- **P2 — nice-to-have.** Eligible to defer.

Tie each item to a success metric or non-goal from intake. Items that don't tie back should be challenged.

If everything looks P0, the cut is wrong. Push back.

## Phase 2 — Parent issue + child issues

Each phase becomes one parent issue (the GitHub equivalent of an Epic — a `type:feature` issue with a task list). Each child issue:

- One clear deliverable.
- Independently testable; AC stated, not implied.
- Honors `.repo-meta.yaml` conventions:
  - `type:` label per issue type (Feature / Bug / Task / Chore / Docs).
  - `area:` label(s) per affected package.
  - Optional milestone (suggested but not assigned at create time).
- Rough sizing: XS / S / M / L / XL.

Capture per child:

- Title (imperative).
- Description (problem + what shipping this enables).
- Acceptance criteria (testable, observable).
- Dependencies (other children, by id).
- Touched packages (from package-context.json).
- Estimated size.
- Issue type.

## Phase 3 — Map cross-package dependencies

For each child that touches a package outside the requester's primary area, flag:

- `cross_package_review_recommended: true` — flag for review attention; even solo, you want to be careful here.
- An explicit pre-condition if a public API is being changed (e.g. "shared types updated in @my-app/shared first").

For children that require new external infra (DB tables, env vars, vendor accounts), add explicit prep tasks. Don't bury infra prep inside a feature task.

## Phase 4 — Risks with mitigations

From `design.json` + execution-level:

- Estimate-bust risk — which children are likely 2x larger?
- Schema-evolution risk — breaking changes to shared types or DB?
- Rollout risk — real possibility of user-facing impact during rollout?
- Solo capacity risk — is this scope realistic given time available?

Each risk: description, likelihood, impact, mitigation.

## Phase 5 — Rollout and rollback strategy

- **Rollout shape** — straight ship / feature flag / dark launch / parallel-run-then-cut. Pick one. State the trigger to advance each stage if multi-stage.
- **Rollback plan** — per stage, what's the unwind.
- **Done-on-prod definition** — for GitHub-tracked work, this is closing the parent issue + milestone close. State any additional gate ("metric sampled at 1 week without regression in adjacent packages").

## Phase 6 — Open questions

Promote from intake + design + new ones uncovered while sizing.

## Phase 7 — Write artifacts

### delivery-plan.md

```
# Delivery plan: <slug>

**Recommendation:** ...
**Phases:** P0 (N), P1 (M), P2 (K)
**Total estimated size:** ...
**Generated:** <date>

## Phase P0 — Parent issue: ...
### Child P0-1 — ...
- Description / AC / Dependencies / Touches / Size / Type

## Phase P1 — Parent issue: ...
## Phase P2 — Parent issue: ...

## Cross-package dependencies
## Risks and mitigations
## Rollout strategy
## Rollback plan
## Open questions
```

### delivery-plan.json

```json
{
  "schema_version": "1.0",
  "slug": "...",
  "generated_at": "2026-06-11",
  "phases": [
    {
      "label": "P0",
      "parent_issue": {
        "title": "...",
        "description": "...",
        "issue_type": "Feature",
        "labels": ["type:feature", "area:api"]
      },
      "children": [
        {
          "id": "P0-1",
          "title": "...",
          "description": "...",
          "acceptance_criteria": ["..."],
          "dependencies": [],
          "touches_packages": ["@my-app/api"],
          "size": "M",
          "issue_type": "Task",
          "labels": ["type:task", "area:api"],
          "cross_package_review_recommended": false
        }
      ]
    }
  ],
  "risks": [{"description": "...", "likelihood": "med", "impact": "high", "mitigation": "..."}],
  "rollout": {
    "shape": "feature-flag",
    "stages": [
      {"name": "shadow", "trigger_to_advance": "..."},
      {"name": "10%", "trigger_to_advance": "..."},
      {"name": "100%", "trigger_to_advance": "..."}
    ],
    "rollback_plan": "..."
  },
  "done_on_prod_definition": "...",
  "open_questions": [
    {"id": "Q-001", "question": "...", "owner_to_answer": "...", "needed_by_phase": "before-delivery-plan-approval", "default_if_unanswered": "..."}
  ]
}
```

## Phase 8 — Hand off

Summarize: phase counts, biggest risk, top open questions, offer to chain to `github-issue-creator` (gated by explicit confirmation).

## Edge cases and rules

- **P0 is empty.** Cut is wrong; redo.
- **Everything is XL.** Children too big; decompose.
- **One child touches 5+ packages.** Decompose by package boundary.
- **Aspirational AC.** Reject; require testable observable assertions.
- **No design.json.** Exit with: "Run `solution-design` first."
- **Single-package repo.** No cross-package coordination; the column collapses but the rest of the plan stands.

## Distinguishing from other skills

- **vs. `github-issue-creator`** — that writes issues. This produces the structured plan.
- **vs. `product-management:sprint-planning`** — that plans the next sprint from a backlog. This produces the backlog at initiative-level.

## What this skill never does

- Never writes to GitHub.
- Never assigns milestones at create time (suggests in the plan; assignment is the user's call).
- Never invents acceptance criteria. If they can't be testable, push back.
