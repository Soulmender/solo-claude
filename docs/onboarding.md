# Onboarding — solo-claude

About 10 minutes to set up your machine. About 5 minutes per repo after that.

## Prerequisites

- Claude Code installed and authenticated.
- A GitHub account with access to the repos you'll work in.
- Git, your package manager (`pnpm`, `yarn`, `npm`, `uv`, `poetry`), and a working terminal.

## Step 1 — Add the marketplace

```bash
/plugin marketplace add https://github.com/<your-user>/solo-claude
```

For private or local development, you can clone the repo and add via `file://`:

```bash
git clone https://github.com/<your-user>/solo-claude ~/code/solo-claude
/plugin marketplace add file:///absolute/path/to/solo-claude
```

## Step 2 — Verify the marketplace

```
/plugin marketplace list
```

You should see `solo-claude` in the output.

## Step 3 — Install the plugin

```
/plugin install solo-claude@solo-claude
```

After install, reload:

```
/reload-plugins
```

## Step 4 — Link the team-wide CLAUDE.md

The plugin ships a CLAUDE.md template that should be active in every Claude Code session. Claude Code reads `~/.claude/CLAUDE.md` automatically, so symlink the plugin's copy:

**macOS / Linux:**
```bash
ln -sf <plugin-install-path>/claude-md-templates/solo-CLAUDE.md ~/.claude/CLAUDE.md
```

**PowerShell (Windows, requires admin):**
```powershell
New-Item -ItemType SymbolicLink -Path "$env:USERPROFILE/.claude/CLAUDE.md" -Target "<plugin-install-path>/claude-md-templates/solo-CLAUDE.md"
```

Find `<plugin-install-path>` via `/plugin list` or check `~/.claude/plugins/solo-claude/`.

Future `/plugin update solo-claude` refreshes the template and the symlink follows automatically.

## Step 5 — Authenticate the GitHub MCP

```
/mcp authenticate github
```

A browser window opens. Sign in. Authorize the scopes (issues + pull requests; read+write).

You'll redo this on each new machine. If the token expires, re-run the same command.

## Step 6 — Verify

In a new Claude Code session:

```
/plugin list
/skill list           # should list all 24 solo-claude skills
/agent list           # currently empty for this marketplace
```

## Step 7 — Onboard your first repo

In a repo you want to use solo-claude on:

```
/scaffold
```

This:

1. Reads `git remote get-url origin` to fill GitHub coordinates.
2. Detects monorepo layout from `pnpm-workspace.yaml`, `package.json` workspaces, `turbo.json`, `nx.json`, `pyproject.toml`.
3. Detects docs layout from filesystem.
4. Reads existing tags to infer versioning.
5. Pulls labels from GitHub.
6. Asks for the rest.

Output: `.repo-meta.yaml` at the repo root. Now every other skill in this plugin reads it for context.

## Step 8 — Try a skill

```
/docs-check                          # 30 seconds — gives you a doc health headline
/review                              # reviews your current branch vs main
/hygiene                             # digest of your open issues / PRs
/plan add a CSV import to the web app   # full planning chain on a vague brief
```

## Daily setup

Whenever you suspect updates:

```bash
/plugin marketplace update
/plugin update
/reload-plugins
```

## Common gotchas

**Skill not triggering.** Skill descriptions are what Claude matches against. Phrasing matters. Nudge by naming it: "use the `repo-audit` skill on this repo."

**MCP says re-authenticate every session.** Re-run `/mcp authenticate github`. Tokens stored in keychain should persist.

**`/plugin install` can't find the plugin.** Check `/plugin marketplace list` — the marketplace name is `solo-claude`. If absent, the `marketplace add` step didn't take.

**Plugin updated but a skill still behaves like the old version.** Run `/reload-plugins`, then restart Claude Code.

**The audit walks too many files in a huge monorepo.** Audit warns at large packages and offers to scope. Use `/audit packages/<one>` to limit scope.

## Help

Open an issue on the `solo-claude` repo.
