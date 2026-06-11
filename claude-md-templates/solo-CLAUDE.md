# Solo CLAUDE.md

This file gets symlinked to `~/.claude/CLAUDE.md`. It carries conventions and "never do X" rules that apply across all your solo / small-team monorepo projects.

For per-repo specifics (this codebase's quirks, gotchas, tech-debt warnings), put a `CLAUDE.md` at the repo root and Claude will read both.

## How you work

You're a solo developer or part of a small team. You ship from `main` more often than not. You use feature branches and PRs for review (your own or a teammate's). Your monorepo uses pnpm or yarn workspaces — packages live under `packages/`. Tracking lives on GitHub Issues with labels + milestones. Docs live in the repo at `README.md` (front door) and `docs/` (everything else).

## Conventions Claude should follow

**Branches.** Feature branches named `<type>/<slug>` (`feat/oauth-flow`, `fix/null-deref-on-empty-cart`, `chore/bump-deps`). One branch per issue when possible.

**Commits.** Conventional Commits (`feat:`, `fix:`, `chore:`, `docs:`, `refactor:`, `test:`). Body wraps at 72. Include the issue ref in the subject when applicable: `feat(auth): add OAuth callback (#123)`.

**PRs.** Title mirrors the conventional commit subject. Description: what changed, why, how to test, screenshots if UI. Link the closing issue with `Closes #123`.

**Issues.** Type label (`type:feature`, `type:bug`, `type:task`, `type:chore`). Area label per affected package (`area:web`, `area:api`, `area:shared`). Milestone = release version. "Done" = issue closed via merged PR.

**Code style.** Whatever the repo's linter says. If unsure, run the linter. Don't invent style.

**Tests.** A change that touches behavior needs a test. A change that touches a public API needs a test. Refactors that change no behavior may not need a test; say so explicitly in the PR.

**Docs.** Public-facing change → README updated in same PR. Architecture-shaping change → `docs/architecture.md` updated. Decisions worth remembering → `docs/adrs/NNNN-<slug>.md`.

## .repo-meta.yaml convention

Each repo using these skills has a `.repo-meta.yaml` at the root. It pins the GitHub owner/repo, the monorepo's package layout, the docs convention, label and milestone conventions, and audit-artifact paths. The skills read it so they don't re-prompt for the same context every run.

Canonical schema: see the marketplace's `docs/repo-meta-yaml-schema.md`.

Generate one for a new repo with the `scaffold-repo-meta` skill.

## .claudedocs/ convention

`.claudedocs/` at repo root is for Claude-generated drafts (audit reports, doc proposals, planning artifacts, scratch). It's gitignored. When a draft is reviewed and ready, promote to `docs/` (or wherever the polished version belongs) and delete the draft.

## Security and "never do X"

- **Never paste credentials, tokens, API keys, or production secrets into chat or into files.** If a secret leaks into a draft, scrub before saving.
- **Never bypass auth or rate-limit checks** in any code you write.
- **Never use string concatenation for SQL.** Parameterize.
- **Never log raw request bodies, raw user input, or raw response bodies that contain PII.** Redact first.
- **Never `eval` or `exec` on external input.** No exceptions.
- **Never use real-looking customer data** in tests, examples, or documentation. Use obviously-fake names (`alice@example.com`, `Acme Corp`).
- **Never commit a `.env` file** or anything in `.env.local`, even if it "only has dev keys."
- **Never push directly to `main`** without a PR, even on personal repos that allow it. The PR creates a record.

## Solo working style — what to keep in mind

- **You're going to forget.** Future-you reads docs and ADRs more than current-you thinks. When you make a non-obvious call, write a short ADR. Two paragraphs is plenty.
- **You don't have a second pair of eyes by default.** The `code-review` skill is your stand-in. Run it before merge.
- **Your CI is your safety net.** Don't bypass it. If a check is flaky, fix it; don't ignore it.
- **Your `main` is your prod (often).** Treat every merge as a deploy candidate. Behind a feature flag if you're not ready.

## What Claude should ask vs. assume

- **Assume** the repo follows the conventions above unless the repo's own `CLAUDE.md` overrides them.
- **Ask** before creating a new top-level folder, a new package, a new GitHub label, or a new milestone — those are durable structural choices.
- **Ask** before deleting any code that's not in `node_modules/`, `dist/`, `.next/`, or an obvious build-output folder.
- **Ask** before changing a config file (CI, package manager, formatter, linter) — easy to break, hard to notice.
