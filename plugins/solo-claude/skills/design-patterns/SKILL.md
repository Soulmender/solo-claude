---
name: design-patterns
description: Catalog the design patterns, idioms, and conventions actually in use across a monorepo or single-package repo — error handling, state management, dependency injection, module boundaries, data access, async patterns, naming, file layout, and testing style. Distinguishes intentional repo-wide patterns from one-off local choices, and flags where a package diverges from the house style. Reads audit.json when present (richer), otherwise scans the filesystem directly. Honors .repo-meta.yaml for layout and package roles. Writes docs/patterns.md plus a patterns.json consumed by docs-author and claude-md-doctor. Use to document "how we do things here", to onboard to an unfamiliar codebase, or before writing a CLAUDE.md. Trigger phrases include "document our design patterns", "what conventions does this repo follow", "catalog the idioms in the codebase", "how is error handling done here", "find where packages diverge from house style".
---

# design-patterns

Extracts the patterns a codebase *actually* follows — not the ones a style guide claims — and writes them down so humans and agents can follow them consistently. Sibling to `repo-audit` (which reports health) and feeder for `docs-author` and `claude-md-doctor`.

## What it produces

- `docs/patterns.md` — human-readable catalog of the repo's design patterns and conventions, grouped by dimension, with concrete file:line examples and a "divergences" section.
- `.claudedocs/patterns/patterns.json` — structured catalog with `schema_version: "1.0"`, consumed by `docs-author` (architecture section) and `claude-md-doctor` (convention source of truth).

Drafts land in `.claudedocs/patterns/` first. Promotion of `patterns.md` to `docs/` is user-gated, same as the audit chain.

## Inputs

- `.repo-meta.yaml` for `monorepo.layout`, `packages`, and `docs` paths. Inferred from filesystem if absent.
- `audit.json` from `repo-audit` if it exists and is fresh — gives the package map, dependency graph, and API surface for free. If absent, the skill does a lighter self-scan.

## Phase 1 — Establish scope and house style

Read `.repo-meta.yaml` and (if present) `audit.json`. Build the package list with roles. Pick a **reference set** — the largest app package plus the most-imported library — as the presumptive "house style". Patterns shared by the reference set are candidate repo-wide conventions; everything else is measured against them.

## Phase 2 — Detect patterns per dimension

For each dimension below, sample across packages (don't read every file; sample proportionally — more from `app`/`library`, less from generated or `tool` packages). Record the dominant approach, the file:line evidence, and any competing approaches.

### Dimensions

- **Error handling** — exceptions vs. result/either types, error classes, error boundaries, how failures cross package boundaries, ret/ log / rethrow conventions.
- **State management** — local vs. global state, store libraries (Redux, Zustand, signals, context), server-state caches (React Query, SWR), immutability conventions.
- **Dependency wiring** — constructor injection, factory functions, service locators, module singletons, DI containers, how config is threaded through.
- **Data access** — ORM vs. query builder vs. raw, repository pattern, where queries live, transaction handling, migration style.
- **Async and concurrency** — promises vs. async/await, cancellation, queues, background jobs, streaming.
- **Module boundaries** — barrel files vs. deep imports, public/internal separation (`index.ts` exports, `internal/` folders), what's allowed to cross package lines.
- **API surface shape** — REST/RPC/GraphQL conventions, request validation (Zod, io-ts, Pydantic), response envelopes, versioning.
- **Naming and file layout** — file naming (kebab/camel/Pascal), folder-per-feature vs. folder-per-type, test-file colocation, component/hook/util conventions.
- **Configuration and env** — how env vars are read and validated, config schemas, secret handling (cross-reference the team CLAUDE.md "never" rules).
- **Testing style** — unit/integration/e2e split, mocking approach, fixture/factory conventions, what's tested vs. skipped, assertion library.
- **Logging and observability** — logger, structured vs. string logs, correlation IDs, what gets logged (and the PII redaction rule from CLAUDE.md).

## Phase 3 — Classify each pattern

For every detected pattern assign:

- **adoption** — `repo-wide` (in the reference set and most packages), `partial` (several packages, not dominant), `local` (one package), or `contested` (two roughly-equal competing approaches — the most useful finding for a CLAUDE.md).
- **intentionality** — `documented` (already stated in a CLAUDE.md / README / ADR), `consistent-undocumented` (clearly a convention, just never written down), or `accidental` (looks incidental).
- **confidence** — `high | medium | low`, based on sample size and consistency.

`contested` and `consistent-undocumented` patterns are the headline outputs: they're exactly what a CLAUDE.md should pin down.

## Phase 4 — Divergences

For each package, list where it departs from the repo-wide patterns, with severity:

- **notable** — a deliberate-looking exception (document the reason if discoverable).
- **drift** — looks like an accident or copy-paste from an older style.
- **legacy** — clearly predates the current convention (git blame age supports this).

Never recommend "fix" here — that's `improvement-advisor`'s job. This skill describes; it does not prescribe remediation.

## Phase 5 — Write artifacts

### patterns.md structure

```
# Design patterns — <repo_name>

**Generated:** <date>
**Commit:** <sha>
**Source:** audit.json (fresh) | direct scan
**Packages sampled:** N

## House style at a glance
One paragraph: the 5-8 conventions a new contributor must internalize.

## Patterns by dimension
### Error handling
- Convention: <dominant approach>  [adoption: repo-wide, confidence: high]
- Evidence: packages/api/src/errors.ts:12, packages/web/src/lib/result.ts:8
- Competing: <if any>

(repeat per dimension)

## Undocumented conventions worth pinning
Bullet list of consistent-undocumented + contested patterns — candidates for CLAUDE.md.

## Divergences by package
### packages/<pkg>
- <dimension>: diverges from house style — <notable|drift|legacy>
```

### patterns.json structure

```json
{
  "schema_version": "1.0",
  "repo_name": "...",
  "generated_at": "2026-06-18",
  "commit": "...",
  "source": "audit-json | direct-scan",
  "reference_set": ["packages/api", "packages/shared"],
  "patterns": [
    {
      "dimension": "error-handling",
      "convention": "Typed Result<T,E> returned from service layer; exceptions only at the HTTP boundary",
      "adoption": "repo-wide",
      "intentionality": "consistent-undocumented",
      "confidence": "high",
      "evidence": ["packages/api/src/errors.ts:12", "packages/shared/src/result.ts:8"],
      "competing": []
    }
  ],
  "divergences": [
    {
      "package": "packages/legacy-admin",
      "dimension": "error-handling",
      "kind": "legacy",
      "detail": "throws raw Error, no Result type",
      "anchor": "packages/legacy-admin/src/users.ts:40"
    }
  ],
  "candidates_for_claude_md": ["error-handling", "module-boundaries", "testing-style"]
}
```

## Edge cases and rules

- **No audit.json.** Proceed with a direct scan; mark `source: direct-scan` and lower confidence ceilings to `medium`. Offer to run `repo-audit` first for a richer result.
- **Single-package repo.** Skip the reference-set comparison; the whole package is the house style. Divergences become intra-package inconsistencies instead.
- **Two genuinely competing patterns at ~50/50.** Report as `contested`; do NOT pick a winner. Surface both with their evidence so the human decides (and records it via `decision-ledger` / a CLAUDE.md edit).
- **Generated packages.** Detect via banner comment; exclude from pattern detection (their idioms are the generator's, not the team's). Note exclusion.
- **Polyglot monorepo** (TS + Python + Go). Catalog per language; don't force one language's patterns onto another. Cross-language conventions (naming, folder layout, commit style) still roll up repo-wide.
- **Very large repo.** Sample, and say so explicitly in the report with the sampling ratio. Never silently sample.

## Distinguishing from other skills

- **vs. `repo-audit`** — audit reports *health* (findings, severity, risks). This skill reports *style* (what the conventions are), with no severity grading. They share the package map; run audit first when you want both.
- **vs. `docs-author`** — docs-author *writes living docs* and will embed this skill's `patterns.json` into `docs/architecture.md`. This skill produces the catalog; docs-author narrates it alongside the architecture.
- **vs. `claude-md-doctor`** — this skill says what the patterns *are*; claude-md-doctor checks whether the CLAUDE.md files *tell agents to follow them*, and repairs the gap.
- **vs. `improvement-advisor`** — this skill never prescribes fixes. Divergences are described, not triaged. Prioritized remediation is improvement-advisor's job.

## What this skill never does

- Never modifies code or CLAUDE.md files. Catalogs only.
- Never grades a pattern as "good" or "bad" — only as adopted/intentional/diverging. Quality judgment belongs to `code-review` and `improvement-advisor`.
- Never auto-promotes `patterns.md` from `.claudedocs/` to `docs/`. User-confirmed promotion only.
- Never invents a convention that isn't evidenced in the code. Every pattern carries file:line evidence or it isn't reported.
