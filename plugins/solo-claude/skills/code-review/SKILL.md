---
name: code-review
description: Review a diff before merging — security, correctness, performance, maintainability, tests, and conventions. Adaptive depth — single-pass for small diffs, fan-out across dimension groups for larger ones. Reads from local git (current branch vs. default branch from .repo-meta.yaml or main). Output grouped by severity (blocker / major / minor / nit) with file plus line anchors and suggested fixes. Optional posting to a GitHub PR via the GitHub MCP, with explicit user confirmation. Use whenever a developer wants a pre-merge sanity check. Trigger phrases include "review my branch", "review this PR", "is this safe to ship", "code review on the changes since main", "do a security review of this diff".
---

# code-review

Reviews a diff against solo-claude conventions and the repo's own CLAUDE.md. Your stand-in second pair of eyes when you don't have one by default.

## What it produces

- An in-chat structured review grouped by severity.
- Optional PR comment posted via the GitHub MCP (only with explicit user confirmation).

No file output by default. Use the `save to file` mode to write the review to `.claudedocs/reviews/<branch>-<short-sha>.md`.

## Phase 1 — Detect what to review

Determine the diff range:

1. If the user provided a PR URL, fetch the PR diff via the GitHub MCP.
2. If the user named a range (`since main`, `vs origin/main`, `last 3 commits`), use it.
3. Otherwise, default to current branch vs. `default_branch` from `.repo-meta.yaml` (fall back to `main`).

For very small diffs (under ~300 changed lines), run a single-pass review. For larger diffs, fan out across dimension groups in parallel.

## Phase 2 — Review dimensions

Three dimension groups, run in parallel for large diffs:

### Group A — Security and correctness
- Auth bypass, missing authz checks
- Injection (SQL, command, template, XSS)
- Secret handling: no hardcoded keys, env vars validated at startup
- Input validation on every public surface
- Error handling: no swallowed exceptions, no `pass` on `except:`
- Race conditions and concurrency hazards
- Null / undefined / empty-list edge cases
- Off-by-one and boundary conditions

### Group B — Performance and maintainability
- N+1 queries (especially around ORM access patterns)
- Sync I/O in async code
- Unbounded loops, unbounded growth of in-memory state
- Dead code, unused imports
- Cyclomatic complexity flags — functions doing too much
- Naming clarity
- Duplication across packages (when the monorepo could share)
- Public APIs without docstrings or types

### Group C — Tests, docs, and conventions
- Test coverage for new behavior (not coverage % — coverage of the new behavior)
- Tests assert behavior, not implementation
- No skipped or `.only` tests left in
- Conventional Commits style for any new commits in the diff
- README updated when public-facing behavior changes
- ADR drafted when a non-trivial architectural decision is in the diff
- Repo's linter rules respected (run if available)

## Phase 3 — Severity classification

Each finding lands in one of:

- **Blocker** — would cause data loss, security breach, or production outage. Do not merge.
- **Major** — significant bug, regression risk, or architectural problem. Address before merge.
- **Minor** — quality issue worth fixing in this PR if cheap, follow-up if not.
- **Nit** — style or preference. Take or leave.

Be honest with severity. Inflated blockers train the user to ignore them; deflated ones miss real risks.

## Phase 4 — Output

Default format in chat:

```
## Code review: <branch> vs <base>

**Verdict:** ship / ship-with-fixes / needs-changes

### Blocker
- `path/to/file.ts:42` — description. Suggested fix: ...

### Major
- ...

### Minor
- ...

### Nit
- ...

### Net
<2-3 sentence summary of the overall shape of the change>
```

If invoked with `quick` mode, run Group A only. If invoked with `full` mode, also run a brief architectural-impact pass (does this change cross a bounded context, introduce a new dependency, change a public API).

## Phase 5 — Optional PR posting

If the user invoked with a PR URL and confirms posting, format the review as a GitHub PR review comment and post via the GitHub MCP. One review per invocation; don't spam multiple comments.

PR posting requires explicit user confirmation each time. Never auto-post.

## Edge cases and rules

- **Diff too large for context.** Sample the changed files by likely-impact (security-sensitive paths first), run Group A on the full diff, run Groups B/C on the sample. Warn the user about the sampling.
- **No changes (empty diff).** Tell the user; offer to switch the base.
- **Mixed Conventional Commit and non-conventional commits in the diff.** Flag in Group C; don't block.
- **The repo has no `.repo-meta.yaml`.** Use sensible defaults; offer to scaffold one after the review.
- **The diff is a generated-code change** (lockfile, schema, OpenAPI client). Run a lightweight version of Group A only; don't review style on generated code.
- **The user is reviewing their own branch pre-PR.** Same review; phrase suggestions as "before you open the PR" rather than "before you merge".

## Distinguishing from other skills

- **vs. `repo-audit`** — audit is full architecture deep-dive on a repo; code-review is a diff-focused pass. Different time horizons (audit: quarterly; review: per PR).
- **vs. `documentation-check`** — that skill checks doc hygiene at the repo level; code-review checks docs only for changed surfaces in this diff.

## What this skill never does

- Never auto-posts to GitHub without confirmation.
- Never inflates severity to look thorough.
- Never reviews dependencies' source code (out of scope; that's audit's job).
- Never modifies files. Suggests fixes inline in the review only.
