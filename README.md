# dev-workflow

Generic agent orchestration plugin for Claude Code. Structured workflow with designer, builder, tester, reviewer, and planner agents — all driven by a single project config file.

## What It Does

- **Router** — Detects intent (build/debug/review/plan), evaluates skip conditions, orchestrates agent chain
- **Designer** — Generates UI code via Gemini MCP, hands off to builder for integration (graceful fallback if no Gemini)
- **Builder** — Implements features with TDD evidence and handoff protocols
- **Tester** — Runs tests, verifies acceptance criteria, produces exit code evidence
- **Reviewer** — Security and pattern validation with confidence scoring
- **Planner** — Creates specs with task breakdowns, validated by Codex dry-run
- **Memory** — Persistent context across sessions (activeContext, patterns, progress)

## Quick Start

### 1. Install the Plugin

```bash
git clone https://github.com/yourusername/claude-dev-workflow.git
```

### 2. Create Project Config

Create `.claude/project.json` in your project root:

```json
{
  "version": "1.0",
  "project": {
    "name": "my-app",
    "description": "My awesome project",
    "tech_stack": ["next.js", "typescript", "postgres"]
  },
  "agents": {
    "tester": { "test_command": "npm test" },
    "skip_conditions": {
      "tester": ["docs-only", "config-only", "user-says-skip"],
      "reviewer": ["docs-only", "config-only", "user-says-skip"],
      "designer": []
    }
  },
  "skills": {
    "project_patterns": ".claude/skills/project-patterns/SKILL.md",
    "auto_load": {
      "ui": ["ui-ux-pro-max"],
      "backend": []
    },
    "work_type_detection": {
      "ui": { "paths": ["components/", "pages/"], "extensions": [".tsx"] },
      "backend": { "paths": ["api/", "server/"], "extensions": [".ts"] }
    }
  },
  "constraints": [
    "Components < 300 lines",
    "Use shadcn/ui components only"
  ],
  "integrations": {}
}
```

### 3. (Optional) Add Project Patterns

Create `.claude/skills/project-patterns/SKILL.md` with your project's conventions. See `templates/project-patterns/SKILL.md` for a starter.

### 4. Add Router Activation to CLAUDE.md

```markdown
**For ANY development task → invoke `dev-workflow-router` skill FIRST. Never bypass.**
Memory: `.claude/memory/activeContext.md` (read on session start)
Config: `.claude/project.json` (project-specific rules)
```

## How It Works

### Agent Chains

| Trigger | Chain |
|---------|-------|
| build (UI) | designer → builder → tester → reviewer → memory |
| build (non-UI) | builder → tester → reviewer → memory |
| error, bug, fix | builder → tester → reviewer → memory |
| review, audit | reviewer → memory |
| plan, design | planner → codex-validate → memory |

### Skip Conditions

Agents can be skipped via `config.agents.skip_conditions`:
- `docs-only` — all changes are documentation files
- `config-only` — all changes are config files (no logic)
- `user-says-skip` — user explicitly says "skip tests" or "quick fix"

When skipped, router produces a `SKIP_REPORT` that replaces the agent's artifact for downstream validation.

### Quality Gates

| Agent | Must Produce | Skippable? |
|-------|-------------|------------|
| Designer | `HANDOFF: BUILDER` + generated code | Yes (no Gemini MCP) |
| Builder | `HANDOFF: TESTER` + changes list | No |
| Tester | `TEST_REPORT: PASS/FAIL/MANUAL` | Yes (via config) |
| Reviewer | `FINAL_STATUS: APPROVED/CHANGES_REQUESTED` | Yes (via config) |
| Planner | Specification + Tasks + Confidence | No |

### Designer Agent (Gemini MCP)

For UI tasks, the designer agent:
1. Generates component code via Gemini MCP
2. Validates against project constraints
3. Hands off to builder for integration with codebase

**Fallback:** If Gemini MCP is unavailable, router skips designer and builder handles everything.

### Memory System

Three files in `.claude/memory/` persist across sessions:

| File | Purpose |
|------|---------|
| `activeContext.md` | Current focus, decisions, learnings |
| `patterns.md` | Architecture patterns, gotchas |
| `progress.md` | Task tracking, verification evidence |

Auto-created on first run.

## Project Structure

```
claude-dev-workflow/
├── .claude-plugin/
│   └── plugin.json              # Plugin manifest
├── agents/
│   ├── designer.md              # UI generation (Gemini MCP)
│   ├── builder.md               # Implementation specialist
│   ├── tester.md                # Quality assurance
│   ├── reviewer.md              # Security & pattern validation
│   └── planner.md               # Architecture & planning
├── skills/
│   ├── dev-workflow-router/
│   │   └── SKILL.md             # Entry point & orchestration
│   ├── dev-workflow-memory/
│   │   └── SKILL.md             # Memory management patterns
│   └── dev-workflow-plan/
│       └── SKILL.md             # Manual plan + codex validation
├── templates/
│   ├── project.json             # Example project config
│   └── project-patterns/
│       └── SKILL.md             # Example project patterns
├── CLAUDE.md                    # Router activation (3 lines)
└── README.md                    # This file
```
