# AGENTS.md — {{PROJECT_NAME}}

> {{PROJECT_DESCRIPTION}}

## Entry Point

**For ANY development task → invoke `dev-workflow-router` skill FIRST. Never bypass.**

If `.claude/project.json` is missing → router triggers `/dev-workflow-init` automatically.

## Available Commands

| Command | When |
|---------|------|
| `/dev-workflow-router` | Any dev task (auto-triggered) |
| `/dev-workflow-init` | Bootstrap project / search skills |
| `/dev-workflow-scan` | Auto-generate patterns from codebase |
| `/dev-workflow-sprint` | Parallel build with agent teams |
| `/dev-workflow-plan` | Plan + Codex validation |
| `/dev-workflow-design` | UI design iteration |
| `/dev-workflow-hotfix` | Production fast-path |
| `/dev-workflow-status` | View progress + runs |
| `/dev-workflow-doctor` | Health check |
| `/dev-workflow-pr` | Generate PR |

## Tech Stack

{{TECH_STACK}}

## Project Structure

```
.claude/
├── project.json          # Project config (source of truth)
├── memory/               # Session memory (auto-managed)
│   └── activeContext.md   # Current context
└── skills/               # Installed skills
    └── project-patterns/  # Project-specific conventions
```

## Key Files

- **Config:** `.claude/project.json` — project rules, constraints, agent settings
- **Memory:** `.claude/memory/activeContext.md` — read on session start
- **Patterns:** `.claude/skills/project-patterns/SKILL.md` — project conventions

## Workflow

1. Router reads config + memory
2. Detects work type (UI, backend, bug, docs, etc.)
3. Loads relevant skills for detected stack
4. Executes agent chain: **Coder → Tester → Reviewer**
5. Logs run + updates memory

## Constraints

{{CONSTRAINTS}}
