# claude-config

My personal Claude setup: working preferences and skills for building apps end to end at Electrica (React + Vite, Java Spring Boot with hexagonal architecture, PostgreSQL, GitHub).

This repo is the single source of truth. Edit files here first, then sync them to claude.ai and Claude Code (see below).

## Contents

| Path | What it is | Used by |
|---|---|---|
| `CLAUDE.md` | Working preferences: straight-to-the-point answers, ask before building, act as a senior coworker | Claude Code (`~/.claude/CLAUDE.md`) |
| `working-preferences.md` | Same text as `CLAUDE.md` | claude.ai / desktop app (Settings → Profile) |
| `skills/app-architect/` | Architecture docs: high-level design, APIs, storage, messaging, trade-offs, detailed design | claude.ai + Claude Code |
| `skills/github-ticket-planner/` | Breaks work into milestones, epics and GitHub Issues; creates nothing without my review and approval | claude.ai + Claude Code |
| `skills/electrica-ui-designer/` | Clickable mockups in Electrica branding, with design tokens and a component inventory | claude.ai + Claude Code |
| `skills/spring-backend-coder/` | Backend coding style: strict hexagonal (ports), SOLID, records, hand-written mappers, tests for every branch | claude.ai + Claude Code |

Keep `CLAUDE.md` and `working-preferences.md` identical.

## Setup

### claude.ai and the Claude desktop app
1. **Preferences:** copy the text of `working-preferences.md` into **Settings → Profile** (personal preferences).
2. **Skills:** in **Settings → Capabilities → Skills**, upload each folder under `skills/` (zip each folder separately). Make sure Skills are turned on.

The skills then work in every chat. Claude picks one automatically when the request matches, or you can name it ("use the architect skill").

### Claude Code (Mac)
Link the files from this repo so that pulling it updates Claude Code automatically:

```bash
# from the root of this repo
mkdir -p ~/.claude/skills

ln -sf "$PWD/CLAUDE.md" ~/.claude/CLAUDE.md

for s in skills/*/; do
  name=$(basename "$s")
  ln -sfn "$PWD/skills/$name" ~/.claude/skills/"$name"
done
```

If you already have a `~/.claude/CLAUDE.md`, back it up first, since `ln -sf` replaces it.

Check that it worked: `ls -la ~/.claude ~/.claude/skills`.

## Updating
1. Edit the file in this repo and commit.
2. Claude Code picks it up immediately (symlinks).
3. claude.ai: re-upload the changed skill in **Settings → Capabilities → Skills**, or paste the updated preferences into **Settings → Profile**.

## Typical flow for a new project
1. **app-architect:** "Design the architecture for …" → architecture document + "Ready for planning" list
2. **github-ticket-planner:** "Plan the tickets in owner/repo" → review document → approve → issues created
3. **electrica-ui-designer:** "Make mockups for …" → clickable, shareable prototype
4. **spring-backend-coder:** implement the backend issues, in Claude Code inside the project repo

Each project repo should also have its own `CLAUDE.md` with project-specific rules (commands, conventions, things not to touch).

## Skill format
Each skill is a folder with a `SKILL.md`:

```markdown
---
name: skill-name
description: One sentence saying when to use it. Claude reads this to decide when to load the skill.
---

# Instructions...
```
