---
name: decision-ledger
description: Capture, query, and apply decisions about audit findings or design choices so they show up annotated (never suppressed) in future audits, verifications, and planning runs. Three modes — extract decisions from a PR or GitHub issue or ADR by reading comments and bodies, record manually from a verbal description, or query the existing ledger for a finding pattern. Each decision is one of known_issue_deferred, intentional_tradeoff, false_positive, wont_fix, or fixed_elsewhere, with rationale, source link, decided_by, decided_at, and an optional expires_at for re-evaluation. Stored per repo in .decisions.yaml at the root. Use after merging a PR that decides something audit-relevant, after closing an issue with a clarifying explanation, or to remember why a finding is intentional. Trigger phrases include "record this decision", "extract decisions from PR 42", "remember why we left this", "log a decision", "query the ledger".
---

# decision-ledger

Compounding memory for the team's decisions. Audits and verifications consult it to annotate matching findings.

## What it produces

`.decisions.yaml` at the repo root, with `schema_version: "1.0"`. Each entry is one decision.

## Three modes

### Mode A — Extract

Reads a source (PR, GitHub issue, ADR, or external doc) for rationale-bearing comments and proposes decision entries.

User invocation: "extract decisions from PR 42", "extract from issue 123", "extract from this ADR".

Walks:

- PR body and review comments.
- Issue body and comments.
- ADR's `Decision` and `Consequences` sections if `Status: Accepted`.

For each candidate rationale (a comment or paragraph explaining "we did X because Y"), propose a decision entry. Show the proposed entries to the user; create them only on confirmation.

Defaults:

- Resolved PR review comments only (unless override).
- ADR must be `Status: Accepted`.
- Filter to authors matching a configurable allowlist (defaults to all; user can set `--authors`).

### Mode B — Record manually

User describes the decision verbally; skill structures and records it.

User invocation: "record a decision: we accept that the auth flow uses session cookies because OAuth 2.0 adoption is post-MVP".

Asks for:

- Category (one of the five).
- Rationale.
- Source (a PR / issue / ADR URL, or "verbal").
- Decided by, decided at, expires at (optional).
- Finding pattern this applies to (so future audits can match).

Writes after confirmation.

### Mode C — Query

User asks: "do we have a decision about <topic>?" or "show recent decisions".

Searches `.decisions.yaml` for matching `finding_pattern` keywords, categories, or anchors. Returns matches with their status and source links.

## Decision schema

```yaml
schema_version: "1.0"
decisions:
  - id: D-001
    finding_pattern:
      category: validation_gap
      anchors:
        - "packages/api/src/orders.ts:42"
      keywords:
        - "POST /orders"
        - "missing zod schema"
    category: false_positive
    rationale: "..."
    source:
      type: github_pr | github_issue | adr | external_doc | verbal | meeting
      ref: "#42"
      url: "https://github.com/owner/repo/pull/42#discussion_r123"
      author: "PeterZnuderl"
      excerpt: "..."
    decided_by: "Peter"
    decided_at: "2026-06-11"
    expires_at: null
    supersedes: null
```

### Categories

- **known_issue_deferred** — we know about it; we'll fix it later (often paired with `expires_at`).
- **intentional_tradeoff** — we chose this consciously; the audit's finding is correct but the trade-off is accepted.
- **false_positive** — the audit's finding is wrong (e.g. tool reports a vuln in a path that's not actually reachable).
- **wont_fix** — we will not fix this; the finding is correct but out of scope forever.
- **fixed_elsewhere** — the finding is real but fixed in a related repo / package / PR.

## Matching rules

When audits / verifications consult `.decisions.yaml` for annotations, they match by:

- **Category overlap** — both finding and decision share a category (validation_gap, security_risk, etc.).
- **Anchor overlap** — at least one file:line anchor in common.
- **Keyword overlap** — at least one keyword in common.

A match requires a minimum composite score (see `references/matching-rules.md` if present, or default: any two of the three).

## Phase summary

Extract / record / query each have a single phase: read input, propose action, confirm, write.

For **extract**, show proposed entries in a numbered list, let user approve, edit, or skip per entry.

For **record**, fill the schema as the user dictates; confirm before write.

For **query**, render matches with their categories, rationale, source link, age, and `expires_at` status (red if expired).

## Edge cases and rules

- **No `.decisions.yaml`.** Create it on first record with the schema header.
- **Conflicting decisions** (two decisions about the same finding with different categories). Surface both; ask the user to add a `supersedes` field.
- **Decisions older than 12 months.** Show with a "stale" marker when surfaced in audits. Don't auto-prune.
- **Sensitive content in rationale or excerpt.** Redact customer identifiers per team CLAUDE.md before writing.
- **A decision references a PR that no longer exists.** Keep the decision; flag the broken link.
- **Bulk extraction across many sources.** Process sequentially; show progress; checkpoint to `.decisions.yaml` after each confirmed entry so partial runs aren't lost.

## Distinguishing from other skills

- **vs. ADRs** — ADRs are formal documents the team writes once. Decisions are smaller, frequent annotations tied to findings. ADRs answer "what did we choose"; decisions answer "why does this finding not need fixing".
- **vs. `repo-verifier`** — verifier reads decisions to annotate; this skill writes them.

## What this skill never does

- Never suppresses findings — only annotates.
- Never overwrites a decision; updates by adding `supersedes`.
- Never invents rationale. If a source comment is too thin, ask the user to elaborate.
