---
name: solution-design
description: Generate two or three candidate architectures for the planned work, pressure-test each against solo-friendly standards (bounded responsibility within the monorepo, deploy independence, reuse vs duplication, complexity, fit with existing patterns), and recommend one with clear rationale. Flags non-trivial decisions that warrant an ADR draft. Produces a high-level component sketch and a comparison table. Use after package-context-scan and before delivery-plan. Trigger phrases include "design the solution", "what is the right shape for this", "compare approaches", "should this be a new package or a module", "draft the design". Reads intake.json plus package-context.json. Writes design.md and design.json. Schema version 1.0.
---

# solution-design

Third phase of `plan-new-work`. Pressure-tests candidate solutions against the monorepo's existing shape.

## What it produces

- `.claudedocs/<slug>/design.md`
- `.claudedocs/<slug>/design.json` — `schema_version: "1.0"`.

## Inputs

- `.claudedocs/<slug>/intake.json` (required).
- `.claudedocs/<slug>/package-context.json` (required).

Exit if either is missing.

## Phase 1 — Generate candidate approaches

Produce 2–3 candidates that genuinely differ in shape. Branch on `work_type`:

### For new-package
- **C1 — New workspace package** (the apparent default).
- **C2 — Add as a module inside the most-relevant existing package.**
- **C3 — Extend the shared library so the capability is available everywhere.**

For monorepos, C2 often wins for small additions; the bar for a new package is "has its own lifecycle or is reused by multiple existing packages." Pressure-test seriously.

### For new-module / package-expansion
- **C1 — Module inside the host package** (default).
- **C2 — New workspace package** (when the module's lifecycle diverges sharply).
- **C3 — Extend a shared library** (when the logic is reusable).

### For library-extension
- **C1 — Extend the existing library** (default).
- **C2 — New companion library** (when the new capability is logically separate).
- **C3 — Push the capability into the consuming package** (when central enforcement isn't needed).

## Phase 2 — Pressure-test each candidate

Evaluate each candidate on:

| Dimension | What to capture |
|---|---|
| **Bounded responsibility** | Does this candidate respect the existing package boundaries? Where does responsibility creep? |
| **Deploy independence** | Can this candidate ship without coordinated changes across packages? For solo dev, this matters for blast-radius and rollback simplicity. |
| **Reuse vs duplication** | What existing package / library does this reuse? Where does it duplicate something we already have? |
| **Operational complexity delta** | New DB table? New env var? New external service? New cron? More moving parts than the problem warrants? |
| **Performance / SLA fit** | Does this candidate plausibly meet the SLA from intake? |
| **Migration cost** | If existing consumers are affected, what's the migration shape? |
| **Backout cost** | If this turns out wrong in 6 months, how hard is it to unwind? |
| **Solo-time fit** | Is the work scoped for solo execution within a reasonable horizon? Or does it need help / external services / time off? Be honest. |

## Phase 3 — Recommend

Pick one candidate. Write 2–4 sentence rationale tied to specific dimensions. The recommendation must explain why the alternatives lose, not just why the chosen one wins.

If genuinely a coin flip, say so and ask the user to break the tie.

## Phase 4 — Identify ADR-worthy decisions

Flag a decision as ADR-worthy if any of:

- Crosses a package boundary (new package, new shared library, new event flow).
- Introduces a new technology, vendor, or runtime.
- Chooses sync vs. async at a load-bearing point.
- Commits to a data-ownership pattern.
- Chooses a release / migration strategy that future-you will need to follow.

For each ADR-worthy decision:

- Title (present tense).
- Context.
- Options (the candidates distilled to this one axis).
- Decision.
- Consequences (upside + follow-on cost).

The orchestrator's downstream ADR-draft step materializes these into `docs/adrs/NNNN-*.md`.

## Phase 5 — Component sketch

Mermaid diagram of packages and major edges affected. Include only what's load-bearing — don't try to redraw the whole monorepo.

```
flowchart LR
  web[packages/web] --> api[packages/api]
  api -- new export 'reserve' --> reservations[packages/reservations NEW]
  reservations --> db[(postgres)]
```

Skip the diagram if it doesn't help.

## Phase 6 — Risks and unknowns

- **Risks** — concrete things that could go wrong.
- **Unknowns** — questions the recommendation depends on that aren't yet answered.

## Phase 7 — Write artifacts

### design.md

```
# Design: <slug>

**Recommendation:** C# — one-liner
**Generated:** <date>

## Problem recap (from intake)
## Candidate approaches
### C1 — ...
### C2 — ...
### C3 — ...
## Comparison table
## Recommendation and rationale
## ADR-worthy decisions
## Component sketch
## Risks and unknowns
```

### design.json

```json
{
  "schema_version": "1.0",
  "slug": "...",
  "generated_at": "2026-06-11",
  "candidates": [
    {
      "id": "C1",
      "label": "...",
      "shape": "...",
      "evaluation": {
        "bounded_responsibility": "...",
        "deploy_independence": "...",
        "reuse_vs_duplication": "...",
        "operational_complexity_delta": "...",
        "sla_fit": "...",
        "migration_cost": "...",
        "backout_cost": "...",
        "solo_time_fit": "..."
      }
    }
  ],
  "recommendation": {
    "candidate_id": "C1",
    "rationale": "...",
    "rejected_alternatives": [
      {"candidate_id": "C2", "reason": "..."}
    ]
  },
  "adr_worthy_decisions": [
    {"title": "...", "context": "...", "options_summary": "...", "decision": "...", "consequences": "..."}
  ],
  "component_sketch_mermaid": "flowchart LR\n  ...",
  "risks": [{"description": "...", "likelihood": "low|med|high", "impact": "low|med|high"}],
  "unknowns": [{"question": "...", "needs_answer_by_phase": "design | delivery-plan | execution"}]
}
```

## Phase 8 — Hand off

Tell the user: recommendation, top risk, ADR count if approved. Offer `delivery-plan` next.

## Edge cases and rules

- **Only one viable candidate.** OK; document why the obvious alternatives were ruled out.
- **User pushes back on the recommendation.** Re-run Phase 3 with their input; capture as "user override".
- **Insufficient package context.** If package-context has more `missing` than `complete`, surface as a confidence caveat.
- **Single-package repo, single-module candidate.** No real comparison needed; produce one candidate with a "considered alternatives" note.
- **Recommendation is "do nothing" or "do later".** Valid output. State the trigger that would re-open the decision.

## Distinguishing from other skills

- **vs. `engineering:system-design`** — that's a general-purpose helper. This skill is opinionated to monorepo + GitHub + in-repo-docs and feeds delivery-plan / ADR-draft in the same chain.
- **vs. `engineering:architecture`** — that skill writes ADRs directly. This flags ADR-worthy decisions and lets the orchestrator materialize them.

## What this skill never does

- Never recommends without rationale.
- Never ignores `package-context.json`.
- Never writes ADRs or other doc files. Flags decisions; downstream skills materialize.
- Never overpromises SLA fit. If uncertain, say so.
