---
name: improvement-advisor
description: Turn findings about a repo into a prioritized, actionable improvement backlog. Consumes audit.json, patterns.json, and verification.json when present (otherwise does a lighter scan), then scores each candidate improvement by impact and effort, sequences them into quick-wins / strategic / long-haul tiers, and respects dependencies between them. Covers refactors, test-coverage gaps, dependency upgrades, pattern-divergence cleanup, architecture risks, and tech-debt paydown. Consults .decisions.yaml so already-decided items are annotated, not re-surfaced. Optionally hands the top items to github-issue-creator as ready-to-file issues. Writes docs/improvements.md plus improvements.json. Use when you want "what should I fix next", a refactor roadmap, or a debt-paydown plan. Trigger phrases include "what should I improve", "suggest improvements", "build a refactor roadmap", "prioritize the tech debt", "turn the audit into a backlog", "what are the quick wins".
---

# improvement-advisor

The prescriptive end of the audit chain. `repo-audit` and `design-patterns` describe what *is*; this skill recommends what to *change*, in what order, and why it's worth it.

## What it produces

- `docs/improvements.md` — prioritized improvement backlog with tiers, per-item impact/effort/rationale, and a suggested sequence.
- `.claudedocs/improvements/improvements.json` — structured backlog with `schema_version: "1.0"`, suitable as input to `github-issue-creator`.
- Optional: filed GitHub issues, via handoff to `github-issue-creator` (opt-in, never automatic).

Drafts land in `.claudedocs/improvements/` first; promotion of `improvements.md` to `docs/` is user-gated.

## Inputs

- `audit.json` (`repo-audit`) — findings with severity and anchors. Primary source.
- `patterns.json` (`design-patterns`) — divergences from house style become cleanup candidates.
- `verification.json` (`repo-verifier`) — false-Done items and unmet requirements become candidates.
- `.decisions.yaml` — recorded decisions, so settled items don't re-surface as new work.
- `quality-history.json` (`repo-audit`) — the quality trend. A metric that is actively *worsening* (coverage falling, findings climbing) raises the priority of items that would reverse it.
- `.repo-meta.yaml` — package roles weight impact (a bug in an `app` outranks the same in a `tool`).

If none of the JSON inputs exist, do a lighter direct scan and say so; recommend running `repo-audit` + `design-patterns` first for a stronger result.

## Phase 1 — Gather candidates

Pull candidate improvements from every available source:

- Audit findings (`major`/`minor` — `blocker`s are not "improvements", they're emergencies; list them in a separate **Do first** callout).
- Pattern divergences classified `drift` or `legacy`.
- Coverage gaps (packages/modules with no or thin tests, weighted by role).
- Dependency health (deeply outdated, deprecated, abandoned, security-flagged).
- Architecture risks (cycles, boundary violations, fan-in/fan-out hotspots).
- Documentation and CLAUDE.md gaps (light pointer; defer detail to `documentation-check` / `claude-md-doctor`).

Deduplicate: one underlying problem surfaced by two sources is one candidate with two evidence links.

## Phase 2 — Score impact and effort

Each candidate gets:

- **impact** `high | medium | low` — blast radius × likelihood of causing a real problem. Weighted by package role and fan-in (a flaw in a package everything imports is higher impact).
- **effort** `S | M | L | XL` — rough size. Note what makes it large (cross-package, needs a migration, touches a hot path, no test safety net).
- **risk-if-deferred** `high | medium | low` — does this get worse / more expensive over time (compounding debt, security exposure) or is it stable?
- **confidence** `high | medium | low` — how sure the recommendation is, given the evidence.

Be explicit about uncertainty. A low-confidence XL recommendation should say "spike first" rather than "do this".

## Phase 3 — Dependencies and sequencing

Build a small dependency graph between improvements (e.g. "extract the shared error type" unblocks "migrate package error handling"). Then tier:

- **Quick wins** — high or medium impact, S/M effort, low risk. Do these first.
- **Strategic** — high impact, M/L effort. Schedule deliberately; may need an ADR.
- **Long-haul** — L/XL effort or low-confidence. Spike, then decide.
- **Watch** — low impact, stable. Recorded so they're not rediscovered every audit; not scheduled.

Honor dependencies: never sequence a blocked item before its blocker, even if the blocker is lower-tier.

## Phase 4 — Consult `.decisions.yaml`

For each candidate, match against recorded decisions (same rule-based matching as `decision-ledger` / `repo-audit`). A candidate the team already decided to accept-as-is gets annotated and dropped from the active tiers into a **Previously decided** section — never silently omitted.

## Phase 5 — Write artifacts

### improvements.md structure

```
# Improvement backlog — <repo_name>

**Generated:** <date>
**Commit:** <sha>
**Sources:** audit.json, patterns.json, verification.json (list what was available)

## Do first (blockers)
Anything from the audit at blocker severity. Not optional improvements.

## Quick wins
| Item | Impact | Effort | Why now |
...

## Strategic
(same table + a one-paragraph rationale each; flag ADR-worthy items)

## Long-haul
(items needing a spike; state the open question to resolve first)

## Watch
Low-priority, recorded so they aren't rediscovered.

## Previously decided
Annotated items from .decisions.yaml.

## Suggested sequence
Ordered list honoring dependencies — the next ~5 things to do.
```

### improvements.json structure

```json
{
  "schema_version": "1.0",
  "repo_name": "...",
  "generated_at": "2026-06-18",
  "commit": "...",
  "sources": ["audit.json", "patterns.json"],
  "items": [
    {
      "id": "imp-001",
      "title": "Extract shared Result type into @app/shared",
      "tier": "quick-win",
      "impact": "high",
      "effort": "M",
      "risk_if_deferred": "medium",
      "confidence": "high",
      "category": "refactor",
      "evidence": ["packages/api/src/errors.ts:12", "patterns.json#divergences[2]"],
      "blocked_by": [],
      "unblocks": ["imp-004"],
      "suggested_label": "type:chore",
      "suggested_area": "area:shared",
      "annotations": []
    }
  ],
  "sequence": ["imp-001", "imp-002", "imp-004"]
}
```

`suggested_label` / `suggested_area` use the repo's `.repo-meta.yaml` `github.labels` namespaces so a downstream issue lands correctly.

## Phase 6 — Optional GitHub handoff

Offer (never automatic) to hand the top N items to `github-issue-creator`. Each item maps to one issue with title, body (problem → evidence → suggested approach → acceptance), the suggested type/area labels, and `unblocks`/`blocked_by` rendered as issue references. The user picks which items become issues. `github-issue-creator` owns the actual creation and its own confirmation gates.

## Edge cases and rules

- **Only blockers, no improvements.** Say so plainly: "Nothing to prioritize beyond the blockers — fix those, re-audit." Don't manufacture filler work.
- **Stale audit.** If `audit.json` predates significant code changes, warn and offer to re-audit before advising. Stale inputs make the backlog wrong.
- **Everything looks low-effort.** Suspicious — effort is probably underestimated. Re-examine cross-package items and missing test safety nets before publishing all-S estimates.
- **Conflicting recommendations** (two improvements that can't both be done). Surface the conflict explicitly; recommend one with rationale; flag it ADR-worthy.
- **No `.repo-meta.yaml`.** Proceed; impact weighting falls back to heuristics (fan-in, file count) without role data.
- **Decision-ledger schema mismatch.** Warn; annotations may be partial; don't drop items you couldn't match.

## Distinguishing from other skills

- **vs. `repo-audit`** — audit *describes findings as they stand*, no prioritization, never prescribes. This skill *prioritizes and recommends* — impact/effort scoring, sequencing, roadmap. It consumes the audit.
- **vs. `code-review`** — code-review evaluates a specific diff before merge. This skill plans changes that haven't been written.
- **vs. `design-patterns`** — design-patterns lists divergences neutrally. This skill decides which divergences are worth fixing and when.
- **vs. `delivery-plan`** — delivery-plan turns one approved design for *new* work into a phased plan. This skill triages *existing-code* debt into a backlog. delivery-plan is forward-looking build; this is backward-looking cleanup.
- **vs. `github-issue-creator`** — this skill decides *what* should become work; issue-creator *files* it. Clean handoff via `improvements.json`.

## What this skill never does

- Never modifies code or docs (beyond writing its own report). Recommendations only.
- Never files GitHub issues itself — always hands off to `github-issue-creator` with user confirmation.
- Never re-surfaces an item already decided in `.decisions.yaml` as new work; annotates instead.
- Never presents an estimate as certain — every item carries a confidence level.
- Never auto-promotes `improvements.md` from `.claudedocs/`.
