---
name: github-issue-creator
description: Create one or more structured GitHub issues from a verbal description. Single issue or epic (parent issue with task-list of child issues). Searches the repo for similar existing issues first to flag duplicates. Templates per type (Feature / Bug / Task / Chore / Docs). Applies labels (type plus area) and milestone from .repo-meta.yaml. Enforces testable acceptance criteria. Shows the proposed issues in chat and creates them in GitHub only after explicit user approval. Use whenever a PM or developer wants to file work. Trigger phrases include "create an issue for", "file a bug about", "decompose this into issues", "I need to track", "open issues for the OAuth flow work".
---

# github-issue-creator

Creates structured GitHub issues. Single issue or decomposed epic.

## What it produces

- Proposed issues displayed in chat for review.
- After user approval, issues created in GitHub via the GitHub MCP.
- Optionally, a `task list` parent issue with checkbox-linked child issues.

## Phase 1 — Read context

Read `.repo-meta.yaml`'s `github` block to know:

- `owner` and `repo` — where to file.
- `labels` — the legal label set (validate against this; warn before creating new labels).
- `milestone_pattern` — for milestone validation if user specifies one.

If outside a repo (no `.repo-meta.yaml`), ask the user which `owner/repo` to file in.

## Phase 2 — Classify and search

Parse the user's request:

- **Single issue?** A focused, bounded ask ("Add a deprecation banner to the orders page").
- **Decomposed epic?** A larger initiative with multiple deliverables ("We need multi-currency support across the orders flow").
- **Bug report?** A defect ("Empty cart causes 500").

Classify the type and area before filing. Pick from `.repo-meta.yaml`'s declared labels.

Search GitHub for similar open issues (via the GitHub MCP) — keyword-match titles and bodies. If matches exist, show the top 3 with status, ask the user to confirm they want to file a duplicate or close-and-replace.

## Phase 3 — Apply template

Per issue type, use the matching template structure.

### Feature

```
## Problem
<the pain or opportunity>

## Proposed solution
<what we want to build, at a high level>

## Acceptance criteria
- [ ] <testable, observable assertion>
- [ ] <another>
- [ ] <another>

## Out of scope
<what this issue explicitly does not cover>

## Notes
<context, links, screenshots>
```

### Bug

```
## Summary
<one-sentence description>

## Repro steps
1. ...
2. ...

## Expected
<what should happen>

## Actual
<what does happen>

## Environment
<browser / OS / package versions>

## Acceptance criteria
- [ ] Repro steps no longer produce the bug.
- [ ] Regression test added.
```

### Task

```
## Goal
<what gets done>

## Acceptance criteria
- [ ] <testable check>
```

### Chore / Docs

```
## What
<one-line description>

## Why
<short rationale if non-obvious>

## Acceptance criteria
- [ ] <observable result>
```

## Phase 4 — Enforce testable acceptance criteria

For every issue, the AC list must contain assertions a third party could verify. Reject:

- "It works correctly."
- "Users are happy."
- "Performance is improved."

Push the user toward observable phrasings:

- "Operator can do X and observes Y."
- "GET /orders/123 returns 200 within 200ms p95."
- "Test suite includes a regression case for this bug."

If the user resists, file anyway with the soft AC and a note that they're hard to verify. Don't gatekeep harder than that.

## Phase 5 — Apply labels and milestone

Apply:

- One `type:` label (matching the classification).
- One or more `area:` labels (per `.repo-meta.yaml`).
- An optional milestone (user-specified or inferred from the active milestone).
- `good-first-issue` if the user explicitly asked.

Validate against `.repo-meta.yaml`'s `labels`. If a label isn't in the legal set, warn — don't auto-create.

## Phase 6 — Decompose if epic

For an epic-shaped request:

1. Generate the parent issue using the Feature template, with body summarizing the initiative.
2. Generate N child issues, each focused on one deliverable.
3. In the parent body, render a task list referencing the children: `- [ ] #<child-id>` (filled in after children are created).
4. File children first, capture their numbers, then file the parent with the task list referencing them.

## Phase 7 — Review and create

Show all proposed issues to the user as a numbered list with titles + body previews. Ask: "Create these in <owner>/<repo>? Reply 'yes' to create all, or list which ones to skip."

Create only on explicit yes. Don't auto-create. Cleanup on mistake is manual.

After creation, list the new issue URLs.

## Edge cases and rules

- **GitHub MCP not authenticated.** Cannot search for duplicates; cannot create. Tell the user to authenticate.
- **Repo has no labels configured at all.** Warn; offer to create the `.repo-meta.yaml` `labels` block (and the labels themselves on GitHub) via `scaffold-repo-meta`.
- **User wants to create a brand-new label.** Confirm explicitly; warn that proliferation makes filtering harder.
- **A milestone the user names doesn't exist.** Don't auto-create; ask whether to create it or pick an existing one.
- **The user's brief is too vague to file as one issue.** Suggest decomposition; ask which path.
- **Sensitive content.** Per team CLAUDE.md, redact customer identifiers before filing — GitHub Issues are world-readable on public repos.
- **Issue closing PRs.** Honor `Closes #<n>` keyword convention; mention it in the rendered AC list if relevant.

## Distinguishing from other skills

- **vs. `delivery-plan`** — that skill produces an Epic + Story tree as JSON; this skill creates the actual GitHub issues. The orchestrator's downstream step hands `delivery-plan.json` to this skill.
- **vs. `github-issue-implementer`** — that skill consumes an existing issue and implements it. This skill creates new ones.

## What this skill never does

- Never creates issues without explicit user confirmation.
- Never auto-creates new labels.
- Never closes existing issues.
- Never assigns reviewers or assignees without being asked.
