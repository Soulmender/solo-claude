# Usage guide — how to use solo-claude day to day

The marketplace ships 24 skills. They pay off when used in the right order, because several feed each other through their JSON artifacts (`audit.json`, `patterns.json`, `code-map.json`, …). This guide is the intended usage pattern.

## The mental model

Three cadences, plus one rule.

**The rule: the foundation has a dependency order.** `repo-audit` → `design-patterns` → `code-map` produce the JSON that almost everything downstream consumes. Run them out of order and the later skills still work, but they warn and fall back to a shallower direct scan. So establish the foundation first, then derive everything from it.

The three cadences:

1. **Once per repo** — establish the context baseline.
2. **Per change** — the steady-state loop, which keeps the baseline fresh as a byproduct.
3. **Periodically / wired up** — catch what the per-change loop missed as the codebase grows.

## 1. Once per machine

Install the plugin, symlink the global `CLAUDE.md`, authenticate the GitHub MCP. See [`onboarding.md`](onboarding.md).

## 2. Once per repo — establish the context baseline

Run this sequence in order. The first four are the foundation; the rest derive from it and can run in any order once the foundation exists.

```
/scaffold        # writes .repo-meta.yaml, seeds the context_index
/audit           # repo-audit → docs/audit/audit.{md,json} + quality baseline
/patterns        # design-patterns → docs/patterns.md + patterns.json
/code-map        # code-map → docs/code-map.{json,md}

# derived (any order):
/docs            # docs-author → README + architecture.md (with patterns) + runbook + ADR stubs
/flows           # flow-docs → docs/flows/*.md for the top 3–5 critical paths
/glossary        # domain-glossary → docs/glossary.{md,json}
/pkg-context     # package-context-author → packages/<pkg>/CLAUDE.md per package
/claude-md       # claude-md-doctor → make the root CLAUDE.md match what was just discovered
```

After this, an agent opening any file has: global rules, local per-package rules, the architecture, the conventions in use, a queryable symbol map, the critical-path behavior, and the domain vocabulary. That is the goal of the baseline.

## 3. Per change — the steady state (self-maintaining)

```
/plan <brief>         # planning chain: intake → context → design → delivery plan
/implement <issue>    # plan-gated, step-by-step execution; closes the doc loop at the end
/review               # pre-merge review against conventions + the repo's CLAUDE.md
```

Lean on one thing here: `/implement`'s **doc-impact phase** runs after the change and offers to refresh exactly the docs the diff invalidated (the code map, the touched package's context file, a flow doc, an ADR). Accept those prompts and the context stays current without extra effort. This loop is where most time goes, and it keeps the foundation fresh as a byproduct.

## 4. Periodically / wired up — keep it honest as it grows

- **`/docs-check` drift mode** on a git hook or schedule — reports which docs have gone stale because the code under them changed since they were last verified. The safety net for when the per-change loop gets skipped.
- **`/code-map refresh`** — incremental, cheap; run whenever you want the index current.
- **Quarterly `/audit`** — trends against the quality baseline, so you see whether things are improving → feed it to **`/improve`** for a prioritized backlog → optionally `/improve issues` to file the top items.
- **`/claude-md`** — whenever you notice agents drifting from the repo's conventions.
- **`/record-decision` / `/extract-decisions`** — whenever a decision worth remembering is made, so future audits annotate rather than re-surface it.

## Edge cases worth knowing

- **Single-package repo.** Skip `/pkg-context` (nothing to scope). `code-map` and `flows` map internal modules instead of packages. Everything else applies.
- **Very large monorepo.** `code-map` and `flows` sample and warn rather than churn. Scope them per package: `/code-map packages/api`, or pick one flow at a time.
- **You don't want all 24 skills.** The lean path is `/scaffold → /audit → /patterns → /code-map → /docs → /claude-md`, then the per-change loop. Add `flows`, `glossary`, and `pkg-context` only where a path or package is genuinely complex — they're the most expensive to produce and keep current.
- **You skipped the foundation order.** Downstream skills won't fail; they warn and produce a lower-confidence result. Re-run `/audit` → `/patterns` → `/code-map` and regenerate for the richer version.

## The short version

Establish the foundation in dependency order once, let the implement-loop keep it fresh, and run the audit and drift checks periodically to catch what the loop missed.

## Which artifact answers which question

`.repo-meta.yaml`'s `context_index` is the live map, but as a quick reference:

| You want to know… | Read |
|---|---|
| The house rules / "never do X" | `CLAUDE.md` (global + root + per-package) |
| How packages are arranged | `docs/architecture.md` |
| How we write code here (idioms) | `docs/patterns.md` |
| Where a symbol lives / who calls it / blast radius | `docs/code-map.json` |
| How a path behaves end to end | `docs/flows/<path>.md` |
| What a domain term means / the canonical name | `docs/glossary.md` |
| What's intentional about a finding | `.decisions.yaml` |
| What to fix next | `docs/improvements.md` |
