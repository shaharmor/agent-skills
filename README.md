# Agent Skills

A Claude Code plugin marketplace with custom agent skills.

## Skills

| Skill | Description |
|-------|-------------|
| `ado-management` | Azure DevOps CLI reference for work items, pull requests, PR threads, and linking |

## Installation

Add the marketplace and install:

```bash
/plugin marketplace add shaharmor/agent-skills
/plugin install shaharmor@agent-skills
```

Skills are then available as `/shaharmor:ado-management`.

## Configuration

Some skills require settings in your `CLAUDE.md` files. For example, `ado-management` expects Azure DevOps settings like:

```markdown
# Azure DevOps Settings

- Organization: <your-org> (https://dev.azure.com/<your-org>)
- Project: <your-project>
- Area Path: <your-area-path>
```

Add these to either:
- **Project-level** (preferred): `.claude/CLAUDE.md` in your repo
- **User-level** (fallback): `~/.claude/CLAUDE.md`
