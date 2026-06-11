---
name: github-issue-implementer
description: Read a GitHub issue, detect the current repo's monorepo layout and conventions, verify the issue against the repo's actual state, produce a structured implementation plan, wait for the developer to approve, then execute the plan under their supervision. Shows diffs step by step, never auto-commits, never auto-pushes. Hard plan-approval gate before any file is written. Hard pre-commit gate that lets the user revise. Picks up the issue from the current branch name if not specified. Use when picking up a tracked issue to implement. Trigger phrases include "implement issue 123", "let's work on the open issue", "pick up the current branch's issue", "drive this issue end to end".
---

# github-issue-implementer

Drives an issue from "I'm picking this up" to "ready for PR". Hard gates on plan approval and commit.

## What it produces

- A working-tree change set across the relevant packages.
- An implementation plan as `.claudedocs/issues/<issue-id>/plan.md`.
- Optionally a draft PR description.

Never auto-commits. Never auto-pushes. Never opens a PR. All those are explicit user steps after.

## Phase 1 — Read the issue

Resolve the issue:

- Explicit argument: `123` or full URL.
- Inferred from current branch name: `feat/oauth-flow-123` or `fix/123-empty-cart`. Extract the number; verify it exists.
- If neither, ask.

Pull the issue via GitHub MCP. Capture: title, body, AC checklist, labels, milestone, linked PRs/issues, recent comments.

## Phase 2 — Verify the issue against repo reality

Before producing a plan, sanity-check the issue:

- **Repo-fit** — does the issue make sense for this repo? (User might have pointed at the wrong issue or wrong repo.)
- **Already-done check** — search the codebase for evidence that the AC is already satisfied. Sometimes issues linger after being implemented elsewhere.
- **Conflicts** — is there an open PR that already addresses this issue? Or a recent commit?
- **Dependencies** — does the issue reference other issues, ADRs, or RFCs that are blocking?

Surface findings before planning. If "already done" or "blocked", ask user how to proceed.

## Phase 3 — Detect repo context

Read `.repo-meta.yaml`. Determine:

- Monorepo layout — `pnpm-workspaces`, etc.
- Which packages are likely affected by the issue (cross-reference issue's `area:` labels with package list).
- Coding conventions for affected packages (read each package's `CLAUDE.md` if present).

## Phase 4 — Produce a structured plan

Write `.claudedocs/issues/<issue-id>/plan.md` with:

```
# Implementation plan: #<id> — <title>

## Acceptance criteria
- [ ] <from issue, verbatim>

## Affected packages
- packages/<pkg> — <why>

## Approach
<2–4 sentences>

## Step 1 — <short label>
- Files to touch: ...
- What changes: ...
- Why: ...
- Test: ...

## Step 2 — ...

## Tests
<what new tests get added, where>

## Out of scope
<things the issue might suggest but this plan doesn't tackle>

## Open questions
<things the user needs to answer before we proceed>
```

Steps should be sized so each one is independently reviewable. Avoid "Step 1: implement everything."

## Phase 5 — Plan approval gate

**Hard gate.** Show the plan. Ask: "Approve this plan, revise, or stop?"

Do not write any code until the user approves. If they revise, regenerate the plan with their input; re-show.

If approved, capture `plan_approved_at` in `orchestrator-state.json` under the issue's `.claudedocs/` folder.

## Phase 6 — Execute step by step

For each plan step:

1. Make the changes.
2. Show a diff per file touched.
3. Run relevant tests if cheap (single-package test command).
4. Pause; ask: "Continue to next step, revise this step, or stop?"

If a step's tests fail, surface the failure clearly; don't paper over it. The user decides whether to fix in-place, revise the plan, or stop.

After the last step, run the full repo test suite if `package.json` `scripts.test` exists at root. Surface results.

## Phase 7 — Pre-commit gate

**Hard gate.** Show the full diff across all touched files. Show summary: files changed, lines added/removed, tests passing/failing.

Ask: "Commit these changes? You'll write the commit message; I'll suggest one." Suggest a Conventional Commit message based on the issue's type label:

```
feat(<area>): <issue title>

Closes #<id>
```

Wait for user's actual commit message. Do not commit on their behalf. Provide the suggested message; let them run `git commit` themselves.

## Phase 8 — Suggest PR description

Optionally draft a PR description following the repo's `.github/pull_request_template.md` if it exists, otherwise a default template:

```
## What
<one-liner>

## Why
<rationale, link to issue>

## How
<approach summary>

## Testing
<what was tested>

Closes #<id>
```

Write to `.claudedocs/issues/<issue-id>/pr-description.md`. The user pastes into the PR when opening.

## Edge cases and rules

- **Issue is already closed.** Don't proceed; ask whether to reopen, file a follow-up, or fix-forward without an issue.
- **Issue has no AC.** Ask the user to add AC before planning; or accept "implicit AC: do what the title says" with a clear warning.
- **The plan would touch packages the issue's `area:` labels don't mention.** Surface; ask whether to add the labels or revise the plan.
- **Mid-execution, tests fail unexpectedly.** Pause. Don't push through. The user decides next step.
- **The user wants to commit before all steps are done.** Honor. Resume later via state.
- **Repo has no `.repo-meta.yaml`.** Run with inference; offer to scaffold afterward.
- **The user requests skipping the plan gate.** Refuse; this is the safety. Offer to make the plan terse but still gated.

## Distinguishing from other skills

- **vs. `code-review`** — that skill reviews a diff someone else (or you) already wrote. This skill writes the diff.
- **vs. `github-issue-creator`** — that skill files new issues. This consumes existing ones.

## What this skill never does

- Never writes code before plan approval.
- Never commits.
- Never pushes.
- Never opens PRs.
- Never closes issues.
- Never bypasses failing tests.
