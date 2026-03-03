# agent8-skills

Skill documents for Agent8 AI.

## Setup

> **Important:** You must run the following command after cloning. Without this, the pre-commit validation will not work.

```bash
git config core.hooksPath .githooks
```

## Project Structure

```
skills/
  <skill-name>.md
.githooks/
  pre-commit
```

All skill documents are placed in the `skills/` directory. Only `.md` files are allowed.

## Writing a Skill Document

### Filename Rules

- Extension: `.md`
- Length: 1-64 characters (excluding `.md`)
- Format: lowercase alphanumeric with single hyphen separators
- Regex: `^[a-z0-9]+(-[a-z0-9]+)*$`

**Valid examples:** `gameserver-sdk-v2.md`, `environment-terrain.md`

**Invalid examples:** `My_Skill.md`, `--double-hyphen.md`, `UPPER.md`

### Frontmatter

Each skill document must start with YAML frontmatter. Only these fields are recognized:

| Field | Required | Constraints |
|-------|----------|-------------|
| `name` | Yes | Must match the filename (without `.md`), 1-64 characters |
| `description` | Yes | 1-1024 characters, supports multi-line YAML values |
| `license` | No | |
| `compatibility` | No | |
| `metadata` | No | String-to-string map |

Unknown frontmatter fields are ignored.

**Example:**

```yaml
---
name: my-skill
description: A short description of what this skill does.
---
```

The `name` field must exactly match the filename. For example, `my-skill.md` must have `name: my-skill`.

### Body

The content after the frontmatter closing `---` is the skill body. Use the `## Instructions` section to guide the AI on how to use the skill.

Common patterns (see [vibe-starter-3d-character.md](skills/vibe-starter-3d-character.md) for a full example):

1. **Instruct to install a package** — Tell the AI to install the required package before use.
   ```markdown
   1. Ensure the package is installed. If not, run: `bun add vibe-starter-3d`
   ```

2. **Instruct to read documentation from the package** — Point the AI to a docs file inside `node_modules/` so it can follow the detailed instructions.
   ```markdown
   2. Read the documentation file at `node_modules/vibe-starter-3d/docs/read_vibe_starter_3d_character.md` and follow its instructions.
   ```

## Pre-commit Hook

A pre-commit hook validates all staged skill documents before each commit. It checks:

1. Only `.md` files exist in `skills/`
2. Filename format and length
3. Frontmatter `name` matches the filename
4. Frontmatter `description` is within 1-1024 characters
