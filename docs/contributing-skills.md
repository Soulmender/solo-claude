# Contributing skills

Adding a new skill, or modifying an existing one, in `solo-claude`.

## File layout

Each skill lives at `plugins/solo-claude/skills/<skill-name>/`:

```
plugins/solo-claude/skills/<skill-name>/
  SKILL.md              # required
  references/           # optional; pulled in when SKILL.md references them
    something.md
```

Skill name is `kebab-case`. Must match the `name:` in SKILL.md frontmatter exactly.

## SKILL.md frontmatter

```yaml
---
name: skill-name
description: A one-paragraph description. Triggerable language. Phrases like "Use when ...", "Trigger phrases include ...". Maximum 1024 characters. No angle brackets in this field.
---
```

**Validation rules:**

- `name` is kebab-case, matches the folder name.
- `description` is 1024 characters or fewer.
- `description` contains no `<` or `>` characters anywhere. Use prose instead.
- Frontmatter is single-line YAML for each field.

## SKILL.md body

The body explains the skill in detail. Suggested sections:

- **What it produces** — concrete outputs.
- **Trigger phrases** — natural-language examples that should fire the skill.
- **Inputs** — what the skill needs to work well.
- **Phases** — numbered phases of the skill's work. Be specific.
- **Output schema** — JSON shape if applicable.
- **Edge cases and rules** — failure modes and how to handle them.
- **Distinguishing from other skills** — when to pick this one over a related skill.
- **What this skill never does** — explicit non-goals.

The body is the actual "prompt" for the skill. Be concrete; example phrasings help; describe edge cases the user is likely to hit.

## Schema versioning

If the skill produces JSON consumed by another skill, declare `schema_version`. Use `"1.0"` for the first version. Bump rules:

- **Additive field** — no schema bump; document at the field.
- **Renamed field** — bump minor (1.0 → 1.1). Update all consumers in the same PR.
- **Removed or semantically-changed field** — bump major (1.x → 2.0). Loud CHANGELOG note; downstream skills should refuse incompatible majors.

## Slash command (optional)

If the skill is frequently triggered and benefits from a memorable command, add a slash command at `plugins/solo-claude/commands/<name>.md`:

```yaml
---
description: One-line description (max ~80 chars)
argument-hint: "<short hint of what to pass>"
---

# /<name>

Wraps `skill-name`.

```
/<name> example invocation 1
/<name> example invocation 2
```
```

Don't add a slash command if the skill triggers reliably from natural language.

## Validation before commit

Run a quick sanity check:

```bash
# desc length and angle bracket check (Python one-liner)
python3 -c "
import re, sys, glob
ok = True
for f in glob.glob('plugins/solo-claude/skills/*/SKILL.md'):
    txt = open(f).read()
    fm = re.match(r'^---\n(.*?)\n---', txt, re.DOTALL).group(1)
    name = re.search(r'^name:\s*(.+)$', fm, re.MULTILINE).group(1).strip()
    desc = re.search(r'^description:\s*(.+)$', fm, re.MULTILINE).group(1).strip()
    skill_dir = f.split('/')[-2]
    if name != skill_dir:
        print(f'{f}: name {name} != folder {skill_dir}'); ok = False
    if not re.match(r'^[a-z][a-z0-9-]*$', name):
        print(f'{f}: name not kebab-case'); ok = False
    if len(desc) > 1024:
        print(f'{f}: description {len(desc)} > 1024 chars'); ok = False
    if '<' in desc or '>' in desc:
        print(f'{f}: angle brackets in description'); ok = False
sys.exit(0 if ok else 1)
"
```

And validate JSON manifests:

```bash
python3 -c "import json; json.load(open('.claude-plugin/marketplace.json'))" && echo OK
python3 -c "import json; json.load(open('plugins/solo-claude/plugin.json'))" && echo OK
```

## Version bump workflow

1. Branch off `main`.
2. Edit files.
3. Bump plugin version in **both** `plugins/solo-claude/plugin.json` and `.claude-plugin/marketplace.json`.
4. Add a CHANGELOG entry.
5. Open a PR. After merge, tag the release: `git tag v<version>`.

## Common pitfalls

**Adding a new skill but forgetting the plugin.json version bump.** Catalog won't pick it up.

**Angle brackets in the description for examples.** Validation fails. Use prose ("triggered by phrases such as `<example>`" → "triggered by phrases like the examples above").

**Description over 1024 chars.** Trim. The description is for triggering; the body is for the details.

**Slash command's `description:` exceeds chips in the menu UI.** Keep slash command descriptions short; the body has the examples.

**Schema_version not bumped on a breaking JSON change.** Downstream skill silently produces wrong output. The whole point of `schema_version` is to refuse incompatible inputs — use it.
