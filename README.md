# Skills

Claude Code skills by [@folpe](https://github.com/folpe).

## Available skills

| Skill | Description |
|-------|-------------|
| [capture-todo](capture-todo/) | Generic Todoist capture + daily-triage with the GTD-allégé methodology (configurable) |
| [rhetorique-master](rhetorique-master/) | Strategic rhetoric for persuasion, negotiation, and argumentative defense |
| [rvp-positioning](rvp-positioning/) | Product positioning using the Rhetorical Value Proposition framework |

## Install

All skills:
```bash
npx skills add folpe/skills
```

Individual skills:
```bash
npx skills add folpe/skills -s capture-todo
npx skills add folpe/skills -s rhetorique-master
npx skills add folpe/skills -s rvp-positioning
```

## Mobile usage (capture-todo)

Claude Code skills load from `~/.claude/skills/` and only run on desktop. To use `capture-todo` from the mobile Claude app, mirror it as a Claude.ai Project:

1. Open https://claude.ai → **Projects** → **Create Project**
2. Name it "Capture Todo" (or similar)
3. Open the Project → **Settings** → **Custom instructions**
4. Paste the contents of [`capture-todo/SKILL.md`](capture-todo/SKILL.md), starting from the `# capture-todo` line (skip the YAML frontmatter)
5. Make sure your account has the Todoist MCP server connected (Claude.ai → Settings → MCP)
6. Use the Project from the mobile app — dictate or type, activation rules behave the same

The `daily-pull` and `digest` primitives (§5–§6) are intended for a scheduled routine and are not useful in interactive chat mode; the rest works as-is.

When you update the skill locally, remember to re-paste the SKILL.md contents into the Project to keep them in sync.
