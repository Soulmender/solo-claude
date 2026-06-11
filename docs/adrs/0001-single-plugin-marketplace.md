# 0001 — One plugin per marketplace for solo-claude

**Status:** Accepted
**Date:** 2026-06-11
**Deciders:** Peter Žnuderl

## Context

`pro-bit-claude` is a four-plugin marketplace (team-core, team-backend, team-frontend, team-pm) because the team has different roles installing different subsets. `solo-claude` targets a different audience: solo developers and small teams who:

- Work in one monorepo at a time.
- Don't have separate "backend" vs "frontend" people installing different plugins.
- Want minimum onboarding friction.
- Use GitHub for everything (issues, code, often docs).

## Decision

Ship one plugin (`solo-claude`) bundling all seventeen skills. No role-split. One install command. One CLAUDE.md.

## Consequences

**Upside.**

- Single `/plugin install` instead of three.
- No "did you install the right plugin?" friction.
- Skills can freely reference each other without cross-plugin concerns.
- Single CHANGELOG.

**Downside.**

- Anyone who only wants `code-review` still installs all 17 skills.
- Skill triggering quality matters more — bad descriptions mean wrong skill fires.
- Major version bumps cascade across everything; can't independently version subsystems.

If solo-claude grows past ~25 skills, revisit by splitting into role-specific plugins (likely splits: `solo-core` for utility + GitHub, `solo-planning` for the planning chain).

## Alternatives considered

- **Mirror the four-plugin pro-bit shape** — rejected as overkill for solo audience.
- **Two plugins (utility + planning)** — rejected for now; planning chain is heavy but solo devs benefit from it too. Revisit if planning skills sprawl.
