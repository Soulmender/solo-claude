---
name: repo-verifier
description: Cross-check the audit JSON against requirements pulled from closed GitHub issues (with acceptance criteria), repo-committed spec files (PRD.md, RFCs), or pasted-in requirements. Classifies each requirement as implemented, partial, not-implemented, contradicted, unverifiable, or out-of-scope. Surfaces closed-but-not-implemented issues (the solo equivalent of false-Done) and recently-shipped work missing from any spec. Consults .decisions.yaml — a not-implemented classification can still be annotated as intentional. Use after repo-audit when you want a reality check against what the project claims to deliver. Trigger phrases include "verify this against the issues", "check if the closed issues are actually done", "find closed-but-not-implemented work", "validate the spec was implemented".
---

# repo-verifier

Reality-check the audit against requirements from GitHub Issues, in-repo specs, or pasted text.

## What it produces

- `docs/audit/verification.md` — human-readable, leading with closed-but-not-implemented findings.
- `docs/audit/verification.json` — structured, `schema_version: "1.0"`.

Paths honored from `.repo-meta.yaml`'s `audit.verification_md` / `audit.verification_json` if pinned.

## Phase 1 — Inputs

Required: an `audit.json` (recent — run `repo-audit` first if missing). Required: a requirements source — at least one of:

- GitHub Issues, filtered by milestone, label, range, or "all closed in last 90 days". Pulled via GitHub MCP.
- A repo-committed spec file (`PRD.md`, `docs/spec.md`, `docs/proposals/*.md`, ADR files marked `Accepted`).
- Pasted-in requirements text from the user.

If multiple sources, normalize each requirement into a uniform record: `{source, id, title, statement, acceptance_criteria, status}`.

## Phase 2 — Per-requirement cross-check

For each requirement, decide:

- **implemented** — audit findings show concrete evidence of the AC being satisfied (file/line anchors).
- **partial** — some AC satisfied, others not. List which.
- **not-implemented** — no audit evidence. For a closed issue, this is a "closed-but-not-implemented" finding — the solo equivalent of false-Done.
- **contradicted** — audit shows the opposite of what the requirement claims (e.g. "auth uses JWT" but audit found session cookies).
- **unverifiable** — the requirement is too vague to verify, or the audit didn't look at that area.
- **out-of-scope** — the requirement is about a feature the repo explicitly doesn't cover.

Calibrate confidence:

- **High** — direct anchor in the audit; clear AC match.
- **Medium** — inferred match (e.g. AC says "users can log in"; audit found a login route but didn't trace the full flow).
- **Low** — audit didn't deeply inspect this surface; classification is best-guess.

Never claim `high-confidence not-implemented` when the audit's coverage of the relevant area was thin. Calibrate down to `medium` or `unverifiable`.

## Phase 3 — Process signals

Surface workflow-level findings that aren't per-requirement:

- **closed_without_implementation_evidence** — closed issue, no matching audit anchor.
- **shipped_without_issue** — commits in the audit range referencing no issue or a non-existent one.
- **milestone_drift** — open milestone past its due date with most issues still open.
- **stale_in_progress** — issues with `in-progress` label / assignee + recent activity older than 21 days.
- **label_violations** — closed issues missing `type:` or `area:` labels per `.repo-meta.yaml`.
- **acceptance_criteria_missing** — closed issues with no AC stated anywhere (description, comments, linked PR).

## Phase 4 — Consult `.decisions.yaml`

For each finding, attempt to match against recorded decisions. A `closed_without_implementation_evidence` finding can be annotated as `intentional_tradeoff` (e.g. "we closed this because we changed our minds"). The finding still appears in the report — the annotation explains the team's position.

## Phase 5 — Output

### verification.md structure

```
# Verification — <repo_name>

**Generated:** <date>
**Audit:** <path-to-audit.json>
**Requirements sources:** ...
**Headline:** N closed-without-evidence, M partial, K not-implemented, J unverifiable

## Closed-but-not-implemented
- #123 — "..." — no audit evidence of <AC item>. <annotation if any>

## Partial
- ...

## Not implemented (still open / never closed)
- ...

## Contradicted
- ...

## Unverifiable
- ...

## Process signals
- ...
```

### verification.json structure

```json
{
  "schema_version": "1.0",
  "repo_name": "...",
  "generated_at": "2026-06-11",
  "audit_ref": "docs/audit/audit.json",
  "requirements_sources": [...],
  "per_requirement": [
    {
      "source": "github_issue",
      "id": "#123",
      "title": "...",
      "classification": "not-implemented",
      "confidence": "high",
      "ac_breakdown": [
        {"text": "...", "satisfied": false, "evidence": null}
      ],
      "annotations": [...]
    }
  ],
  "process_signals": [
    {"type": "closed_without_implementation_evidence", "id": "#123", "severity": "major"}
  ],
  "totals": {"implemented": 12, "partial": 3, "not_implemented": 5, "contradicted": 0, "unverifiable": 4, "out_of_scope": 1}
}
```

## Phase 6 — Update `.repo-meta.yaml`

Update the `audit:` block with `verification_md`, `verification_json`, `verification_last_run`. Diff before write.

## Edge cases and rules

- **No audit JSON.** Exit with: "Run `repo-audit` first."
- **No requirements source at all.** Ask the user to point at one. Don't synthesize.
- **GitHub MCP not authenticated.** Skip GitHub-issue source; use only repo-committed specs / pasted text.
- **Audit JSON schema version older than 1.0.** Refuse to run; suggest re-audit. (Future-proofing for when this version changes.)
- **An issue has no body and no comments.** AC missing finding; classification is `unverifiable`.
- **A milestone matches `.repo-meta.yaml`'s pattern but no issues are in it.** Skip silently; not a finding.
- **Sensitive content.** Per team CLAUDE.md, redact customer identifiers in pulled issue text before writing reports.

## Distinguishing from other skills

- **vs. `repo-audit`** — audit maps what the code does. Verifier checks what's claimed.
- **vs. `code-review`** — review evaluates a diff. Verifier evaluates the repo as a whole against requirements.
- **vs. `workflow-hygiene-check`** — that skill is daily/weekly issue hygiene. Verifier is per-release / per-audit AC checking.

## What this skill never does

- Never closes or reopens GitHub issues. Recommendations only.
- Never overrides a `Done` classification silently. Surfaces evidence; lets the user decide.
- Never suppresses findings based on annotations.
