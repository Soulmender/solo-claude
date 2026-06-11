---
name: workflow-hygiene-check
description: Daily or weekly digest of unhealthy GitHub Issues states in a repo. Surfaces issues without type or area labels, issues idle for too long, milestone slip (open milestone past its due date), milestone naming violations against the .repo-meta.yaml pattern, PRs idle in review more than 3 business days, draft PRs that have been draft longer than a week, closed issues with no linked PR, and any contributor with too many in-progress items at once. Reads .repo-meta.yaml for the legal label set, milestone pattern, and default branch. Use as a Monday-morning sanity check, before milestone close-out, or whenever the issue tracker feels disorganized. Trigger phrases include "hygiene check on this repo", "what is the state of the backlog", "find stale issues", "any PRs idle in review", "Monday digest of the repo".
---

# workflow-hygiene-check

A digest of unhealthy states in a GitHub-tracked repo. Designed to run quickly so it can be daily/weekly without ceremony.

## What it produces

In-chat structured digest. Optional save to `.claudedocs/hygiene/<date>.md`.

## Phase 1 — Read inputs

Read `.repo-meta.yaml`'s `github` block (owner, repo, legal labels, milestone_pattern, default_branch).

Determine scope:

- **Per-repo** — default, only this repo's issues / PRs.
- **Fleet-wide** — if the user maintains multiple repos and lists them, walk each. Honor only `.repo-meta.yaml`-equipped repos; warn about the rest.

## Phase 2 — Pull state via the GitHub MCP

For each scoped repo, pull:

- Open issues (titles, labels, assignees, last update, linked PRs).
- Open PRs (titles, base, head, state, last review activity).
- Open milestones (due date, open/closed issue counts).
- Recently closed issues (last 30 days, for the linked-PR check).

If the GitHub MCP is not authenticated, exit with: "Authenticate `/mcp` first for the GitHub server."

## Phase 3 — Run checks

### Label hygiene
- Open issues missing a `type:` label → list.
- Open issues missing any `area:` label → list (closed issues are exempt).
- Issues with labels not in `.repo-meta.yaml`'s legal set → list (could indicate label drift or new-but-undeclared categories).

### Issue staleness
- Open issues with no update in greater than 21 days → list, oldest first.
- Open issues that have a `blocked` label and the blocker is stale → list with what they're blocked on.
- Closed issues with no linked PR → list (might indicate "closed without implementation evidence" — solo equivalent of false-Done).

### PR hygiene
- Open PRs awaiting review for more than 3 business days → list.
- Draft PRs that have been draft for more than 7 days → list.
- PRs against a branch that no longer exists → list.

### Milestone hygiene
- Open milestones past their due date → list, with completion percentage.
- Closed milestones with open issues still in them → list.
- Milestone names not matching `.repo-meta.yaml`'s `milestone_pattern` → list.

### Contributor load
- Anyone with 4 or more open issues assigned to themselves → list (likely overloaded).
- Anyone with multiple in-progress PRs at once → list.

(For solo repos, both of these usually point at "you" — useful self-check.)

## Phase 4 — Headline + digest

```
## Hygiene: <repo> — <date>

**Headline:** N findings (B blockers, M major, K minor)

### 🔴 Blockers (action needed)
- Milestone v1.5.0 past due (45/60 closed)

### 🟡 Worth attention
- 7 open issues missing `area:` label
- 3 PRs awaiting review more than 3 days
- 2 issues stale more than 60 days

### 🟢 OK
- Label hygiene: clean
- Contributor load: balanced
```

Headline: red if any milestone is overdue or any PR awaiting review more than 7 days, yellow if any minor finding exists, green otherwise.

## Phase 5 — Offer follow-up

End with a short prompt for next actions:

- "Want me to draft a comment on the stale issues asking for status?"
- "Want me to bulk-add `area:web` to the 5 unlabeled issues?"
- "Run `repo-verifier` on the closed-no-PR issues to see if they're really done?"

Each offer requires explicit confirmation before any state-changing action.

## Edge cases and rules

- **Very large repo (greater than 500 open issues).** Sample by recency + assignee; warn about sampling.
- **GitHub MCP not authenticated.** Exit cleanly.
- **No `.repo-meta.yaml`.** Run with sane defaults; recommend scaffolding.
- **No milestones at all.** Skip the milestone block; not a finding by itself.
- **PRs auto-merged with no review.** Don't flag — that's a separate policy question, not hygiene.
- **CI failure on a PR.** Out of scope here; that's the PR author's responsibility. This skill is about issue/PR workflow, not pipeline health.
- **Bot-created issues / PRs** (Dependabot, Renovate). De-emphasize in the digest; surface only if they're past stale thresholds. They shouldn't dominate the report.

## Distinguishing from other skills

- **vs. `repo-verifier`** — that skill checks code against AC. This checks state hygiene.
- **vs. `documentation-check`** — doc-check looks at docs; this looks at GitHub tracking artifacts.
- **vs. `code-review`** — code-review evaluates a diff. This evaluates the issue/PR queue.

## What this skill never does

- Never modifies issues / PRs. Reports only. Bulk actions are offered as follow-ups and require explicit confirmation.
- Never assigns issues to people.
- Never closes anything.
