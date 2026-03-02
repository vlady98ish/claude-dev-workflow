<p align="center">
  <h1 align="center">shut-up-and-ship</h1>
  <p align="center">
    Stop configuring. Start shipping.<br/>
    Agent orchestration plugin for <a href="https://docs.anthropic.com/en/docs/claude-code">Claude Code</a>
    <br /><br />
    <em>7 agents &middot; 11 skills &middot; config-driven &middot; memory across sessions</em>
  </p>
  <p align="center">
    <a href="#quick-start">Quick Start</a> &middot;
    <a href="#commands">Commands</a> &middot;
    <a href="#how-it-works">How It Works</a> &middot;
    <a href="#configuration">Configuration</a>
  </p>
</p>

---

You describe what to build. Seven agents figure out the rest — a debugger that diagnoses before fixing, a tester that verifies before reviewing, a reviewer that gates before merging. Memory survives between sessions so you never re-explain context.

One config file. Any project. Shut up and ship.

## Quick Start

### Install

```
/plugin marketplace add vlady98ish/claude-ship
/plugin install shut-up-and-ship@shut-up-and-ship
```

### Use

Just start working — the router auto-detects your intent and runs the right workflow:

```
> build a login page          → BUILD:          Designer → Builder → Tester → Reviewer
> implement infinite canvas   → BUILD_COMPLEX:  Planner → Builder → Tester → Reviewer
> fix the auth crash          → DEBUG:          Debugger → Builder → Tester → Reviewer
> plan the payments feature   → PLAN:           Planner → Codex Validate
> review the auth module      → REVIEW:         Reviewer
```

On first run, the plugin bootstraps your project: detects tech stack, installs recommended skills from [skills.sh](https://skills.sh/), and generates config.

## What You Get

**Agents** — 7 specialized roles, each with a strict contract:

| Agent | Does | Produces |
|-------|------|----------|
| Debugger | Root cause analysis before any fix attempt | `DIAGNOSIS` |
| Designer | UI generation via Cursor Agent / Gemini MCP | `DESIGNER_COMPLETE` |
| Builder | Implementation with TDD evidence | `BUILDER_COMPLETE` |
| Tester | Runs tests, maps results to acceptance criteria | `TEST_REPORT` |
| Reviewer | Security + pattern validation (read-only) | `REVIEW_REPORT` |
| Planner | Specs, task breakdown, flow diagrams | `PLAN_COMPLETE` |
| Migrator | DB migrations with rollback plans | `MIGRATION_PLAN` |

**Memory** — context that survives between sessions:

| File | Tracks |
|------|--------|
| `activeContext.md` | Current focus, decisions, learnings |
| `patterns.md` | Architecture conventions, gotchas |
| `progress.md` | Tasks, completed work, test evidence |
| `runs.jsonl` | Every workflow execution (append-only) |

**Features** — optional tracking tools:

| Feature | What It Does | Default |
|---------|-------------|---------|
| Decision Log | Records *why* you chose X over Y — survives compaction | ON |
| Flow Diagrams | Mermaid diagrams of data flows, generated during planning | ON |
| Kanban Board | Visual task board synced from progress (supports ClickUp, GitHub, Linear) | OFF |

## Commands

| Command | When |
|---------|------|
| `/ship-router` | Any dev task (auto-triggered) |
| `/ship-init` | Bootstrap project or search for skills |
| `/ship-plan` | Plan + Codex validation before building |
| `/ship-design` | Iterate on UI before building |
| `/ship-hotfix` | Production incident fast-path |
| `/ship-sprint` | Parallel build with agent teams + worktrees |
| `/ship-scan` | Auto-generate patterns from your codebase |
| `/ship-status` | View progress, recent decisions, run history |
| `/ship-doctor` | Health check: config, memory, MCP, features |
| `/ship-pr` | Generate PR from workflow artifacts |

## How It Works

### The Pipeline

Every request goes through the router, which reads your config, detects intent, and runs the right agent chain:

```
  You: "fix the login bug"
   │
   ▼
┌─────────────────────────────────┐
│  Router                         │
│  1. Load config (project.json)  │
│  2. Detect intent → DEBUG       │
│  3. Load memory (3 files)       │
│  4. Load skills for your stack  │
│  5. Check skip conditions       │
└──────────────┬──────────────────┘
               │
   ┌───────────▼───────────┐
   │  Debugger → Builder → │
   │  Tester → Reviewer    │
   └───────────┬───────────┘
               │
   ┌───────────▼───────────┐
   │  Log run + Update     │
   │  memory + Sync        │
   └───────────────────────┘
```

### Workflow Chains

| Workflow | Chain | Trigger |
|----------|-------|---------|
| **BUILD** | [Designer] → Builder → Tester → Reviewer | Default for any task |
| **BUILD (UI)** | Designer → Builder → Tester → Reviewer | UI-related files detected |
| **BUILD_COMPLEX** | Planner → [Designer] → Builder → Tester → Reviewer | Complex architecture auto-detected |
| **DEBUG** | Debugger → Builder → Tester → Reviewer | "fix", "bug", "error", "broken" |
| **PLAN** | Planner → [OctoCode Research] → [Codex Validate] | "plan", "design", "architect" |
| **HOTFIX** | Debugger → Builder → Tester (targeted) | `/ship-hotfix` |
| **MIGRATE** | Migrator → [Builder] → Tester → Reviewer | "migrate", "schema", "migration" |
| **REVIEW** | Reviewer | "review", "audit", "check" |
| **SPRINT** | Planner → parallel Builders (worktrees) → Merge → Test → Review | `/ship-sprint` |
| **DESIGN** | Designer ↔ User (iterate) | `/ship-design` |

`[agent]` = skippable via config.

### Automatic Retries

Tests fail? Reviewer requests changes? The router retries the builder automatically:

```
Builder → Tester ──PASS──→ Reviewer ──APPROVED──→ Done ✓
              │                         │
            FAIL                  CHANGES_REQUESTED
              │                         │
         Builder (retry)           Builder (retry)
              │                         │
         max 3 attempts            max 2 attempts
```

### Agent Contracts

Every agent ends with a structured JSON contract. The router validates required fields before unblocking the next agent. Missing fields? A remediation task is created automatically.

## Configuration

All config lives in `.claude/project.json`:

```json
{
  "version": "1.0",
  "project": {
    "name": "my-app",
    "tech_stack": ["typescript", "next.js"]
  },
  "agents": {
    "tester": { "test_command": "npm test" },
    "retry_limits": { "tester_fail": 3, "reviewer_changes": 2 },
    "skip_conditions": {
      "tester": ["docs-only", "config-only", "user-says-skip"],
      "reviewer": ["docs-only", "config-only", "user-says-skip"]
    }
  },
  "skills": {
    "project_patterns": ".claude/skills/project-patterns/SKILL.md",
    "auto_load": { "ui": ["ui-ux-pro-max"], "backend": [] }
  },
  "constraints": ["Components < 300 lines"],
  "mcp": {
    "codex": { "enabled": true, "required": false },
    "octocode": { "enabled": true, "required": false }
  },
  "features": {
    "decision_log": { "enabled": true, "path": "docs/decisions/DECISIONS.md" },
    "flow_diagrams": { "enabled": true, "format": "mermaid" },
    "kanban": { "enabled": false, "path": "docs/kanban/BOARD.md", "sync": "none" }
  }
}
```

### Key Config Options

| Key | What It Controls |
|-----|-----------------|
| `agents.tester.test_command` | Command the tester agent runs |
| `agents.skip_conditions` | When to skip tester/reviewer (e.g. docs-only changes) |
| `agents.retry_limits` | Max retries for test failures / review changes |
| `constraints[]` | Freeform rules agents must follow (e.g. "No files > 400 lines") |
| `features.decision_log` | Append-only log of architectural decisions |
| `features.flow_diagrams` | Auto-generate Mermaid flow diagrams during planning |
| `features.kanban` | Task board synced from progress (`sync`: none / github / linear / clickup) |
| `mcp.codex` | Plan validation via Codex subagent (auto-skipped if not installed) |
| `mcp.octocode` | Deep code search & research during planning (auto-skipped if not installed) |

## MCP Servers (Optional)

The plugin works without any MCP servers, but these supercharge specific workflows:

| MCP | What It Does | Used By |
|-----|-------------|---------|
| **OctoCode** | Deep code search — GitHub repos, package APIs, PR history. Better than context7 for code forensics | Planner (research phase) |
| **Codex** | Spawns a subagent to validate plans against your codebase | Router (plan validation) |
| **Gemini** | UI component generation | Designer agent |
| **ClickUp** | Kanban board sync | Router (if kanban.sync = "clickup") |

All are **gracefully optional** — if not installed, the workflow skips that step and continues. The `/ship-doctor` command shows which MCPs are connected.

## Skills & Marketplace

Skills are auto-recommended based on your detected tech stack, installed from [skills.sh](https://skills.sh/):

| Stack | Auto-Recommended |
|-------|-----------------|
| **React Native** | vercel-react-native, ui-ux-pro-max, expo-native-ui, expo-deployment |
| **Next.js** | vercel-react, web-design-guidelines, nextjs-app-router, webapp-testing |
| **Python** | python-testing |
| **Supabase** | supabase-postgres |

Search for more:

```bash
/ship-init search <query>    # inside Claude Code
npx skills find <query>              # from terminal
```

Browse: [skills.sh](https://skills.sh/) · [claudemarketplaces.com](https://claudemarketplaces.com/) · [skillsmp.com](https://skillsmp.com/)

## Project Structure

```
shut-up-and-ship/
├── agents/                        # 7 specialized agents
│   ├── builder.md                 #   Implementation + TDD
│   ├── debugger.md                #   Root cause analysis
│   ├── designer.md                #   UI generation
│   ├── migrator.md                #   DB migrations
│   ├── planner.md                 #   Architecture & specs
│   ├── reviewer.md                #   Security & patterns
│   └── tester.md                  #   Test execution
├── skills/                        # 11 workflow skills
│   ├── ship-router/       #   Entry point & orchestration
│   ├── ship-init/         #   Project bootstrap
│   ├── ship-memory/       #   Session memory management
│   ├── ship-plan/         #   Planning + Codex validation
│   ├── ship-sprint/       #   Parallel builds
│   ├── ship-design/       #   UI design iteration
│   ├── ship-hotfix/       #   Production fast-path
│   ├── ship-scan/         #   Auto-detect patterns
│   ├── ship-status/       #   Progress dashboard
│   ├── ship-doctor/       #   Health diagnostics
│   └── ship-pr/           #   PR automation
├── registry/                      # Skill catalog + bundled skills
│   ├── catalog.json               #   Stack → skill mapping
│   └── skills/                    #   Offline fallbacks
├── templates/                     # Config & doc templates
│   ├── project.json
│   ├── AGENTS.md
│   └── CLAUDE.md
└── .claude-plugin/
    └── plugin.json                # Plugin manifest (v2.0.1)
```

### Per-Project Files (auto-created)

```
your-project/
├── CLAUDE.md                      # Points to AGENTS.md
├── AGENTS.md                      # Full AI agent instructions
├── .claude/
│   ├── project.json               # Project config (source of truth)
│   ├── memory/                    # Persists across sessions
│   │   ├── activeContext.md
│   │   ├── patterns.md
│   │   ├── progress.md
│   │   └── runs.jsonl
│   └── skills/
│       └── project-patterns/      # Your project conventions
└── docs/                          # Generated artifacts
    ├── plans/                     # Feature plans
    ├── decisions/DECISIONS.md     # Decision log
    ├── flows/                     # Mermaid flow diagrams
    └── kanban/BOARD.md            # Task board (if enabled)
```

## License

MIT
