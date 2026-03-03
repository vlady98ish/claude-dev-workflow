---
name: ship-doctor
user-invocable: true
description: |
  Diagnose plugin health: config validation, MCP connections, memory integrity, agent availability.
  Use when: something isn't working, first-time setup, after updates.

  Triggers: /ship-doctor
allowed-tools: Read, Bash, Glob, Grep
---

# Workflow Doctor

**Diagnose and fix plugin health issues.**

## Checks (run all, report results)

### 1) Config Validation
```
Read(file_path=".claude/project.json")
```

Verify:
- [ ] File exists
- [ ] Valid JSON
- [ ] Has `version` field
- [ ] Has `project.name`
- [ ] Has `project.tech_stack` (array)
- [ ] Has `agents.tester.test_command`
- [ ] Has `agents.skip_conditions` (object)
- [ ] Has `constraints` (array)
- [ ] `skills.project_patterns` path exists (if set)
- [ ] All `auto_load` skill paths exist

**Fix suggestions:** Create missing fields with defaults from template.

### 2) Memory Health
```
Bash(command="ls -la .claude/memory/ 2>/dev/null || echo 'MISSING'")
```

Verify:
- [ ] `.claude/memory/` directory exists
- [ ] `activeContext.md` exists and has stable anchors
- [ ] `patterns.md` exists and has stable anchors
- [ ] `progress.md` exists and has stable anchors
- [ ] No file > 50KB (compaction needed)
- [ ] `runs.jsonl` exists (create if missing)

**Check anchors:**
```
Grep(pattern="## Current Focus", path=".claude/memory/activeContext.md")
Grep(pattern="## Common Gotchas", path=".claude/memory/patterns.md")
Grep(pattern="## Tasks", path=".claude/memory/progress.md")
```

**Fix suggestions:** Create missing files from templates. Compact large files.

### 3) MCP Connections

Check each MCP based on `config.mcp` settings:

| MCP | Tool Probe | Used By | Required? |
|-----|-----------|---------|-----------|
| Codex | `ToolSearch(query="codex")` | Plan validation | No (skips if missing) |
| OctoCode | `ToolSearch(query="octocode")` | Deep research in planning | No (skips if missing) |
| Gemini | Check designer backend config | Designer agent | No (falls back to skill-only) |
| ClickUp | `ToolSearch(query="clickup")` | Kanban sync (if configured) | No |

For each:
- [ ] MCP server responds to ToolSearch
- [ ] If `config.mcp.{name}.enabled` but not found → warn with install instructions
- [ ] If `config.mcp.{name}.required` and not found → error

**Fix suggestions:**
- Missing Codex → "Add codex-subagent MCP server to your Claude Code settings"
- Missing OctoCode → "Add octocode MCP server — enables deep code search during planning"
- Missing Gemini → "Designer will use skill-only mode (no MCP generation)"

### 4) Agent Availability

Verify all expected agent files exist:
```
Glob(pattern="agents/*.md")  # In plugin directory
```

Expected agents:
- [ ] builder.md
- [ ] tester.md
- [ ] reviewer.md
- [ ] planner.md
- [ ] designer.md
- [ ] debugger.md
- [ ] migrator.md

### 5) Skills Availability

Verify referenced skills exist:
```
# Check project_patterns path from config
# Check each auto_load skill path
```

### 6) Features Validation

```
Read(file_path=".claude/project.json")  # already loaded from step 1
```

If `features` key exists, validate:
- [ ] `features.decision_log.enabled` is boolean
- [ ] `features.decision_log.path` is string (if enabled)
- [ ] `features.flow_diagrams.enabled` is boolean
- [ ] `features.flow_diagrams.format` is "mermaid" (only supported format)
- [ ] `features.kanban.enabled` is boolean
- [ ] `features.kanban.path` is string (if enabled)
- [ ] `features.kanban.sync` is one of: "none", "github", "linear", "clickup"

If `features` key missing → OK (defaults applied by router).

If `features.decision_log.enabled`:
- [ ] `docs/decisions/DECISIONS.md` exists
- [ ] File contains `## Log` anchor
  ```
  Grep(pattern="## Log", path="docs/decisions/DECISIONS.md")
  ```

If `features.kanban.enabled`:
- [ ] `docs/kanban/BOARD.md` exists
- [ ] File `## Last Updated` timestamp is < 24h old (warn if stale)

**Fix suggestions:**
- Missing DECISIONS.md → "Run /ship-init or create from template"
- Missing `## Log` anchor → "Add `## Log` section to DECISIONS.md"
- Stale kanban → "Next workflow run will refresh it"

### 8) LSP Health

Check LSP configuration from project.json `lsp` section:

```
Read(file_path=".claude/project.json")  # already loaded from step 1
```

**If `lsp` key missing or `lsp.enabled === false`:**
Report as SKIP (not configured). Suggest: "Run `/ship-init` to enable LSP, or add manually to project.json."

**If `lsp.enabled === true`:**

| Check | How | Pass | Fail Fix |
|-------|-----|------|----------|
| `ENABLE_LSP_TOOL` flag | `Read(file_path="~/.claude/settings.json")` — check for `"ENABLE_LSP_TOOL"` in env | ✓ flag set | ✗ Add `"env": {"ENABLE_LSP_TOOL": "1"}` to `~/.claude/settings.json` |
| Language server binary | `Bash(command="which {server_name}")` for each server in `lsp.servers` | ✓ found | ✗ Run: `{server.install}` |
| Claude LSP plugin | `Bash(command="claude plugin list 2>&1")` — check for LSP-related plugin | ✓ enabled | ✗ Run: `claude plugin enable {plugin}` |

**Output format:**
```
### LSP
- Config: ✓ enabled (2 servers configured)
- ENABLE_LSP_TOOL: ✓ set in ~/.claude/settings.json
- typescript-language-server: ✓ installed (/usr/local/bin/typescript-language-server)
- pyright: ✗ NOT installed → Run: npm i -g pyright
- Plugin: ✓ enabled
```

Or if not configured:
```
### LSP
- Config: SKIP (not configured)
  → Enable with /ship-init or add "lsp" section to .claude/project.json
```

---

### 9) Stale State Detection
- Any tasks stuck in `in_progress` for > 24 hours?
- Any `runs.jsonl` entries showing repeated failures?
- Any memory files with `## Last Updated` > 7 days old?

## Output Format

```
## Workflow Doctor Report

### Config ✓
- project.json: valid
- All fields present

### Memory ✓
- 3/3 files present
- All anchors intact
- Sizes OK (activeContext: 12KB, patterns: 8KB, progress: 10KB)

### MCP Connections
- Codex: ✓ available (plan validation)
- OctoCode: ✓ available (deep research)
- Gemini: ✓ available (designer)
- ClickUp: ✗ not configured (optional, for kanban sync)

### Agents ✓
- 7/7 agents present

### Skills ✓
- project-patterns: ✓ found
- ui-ux-pro-max: ✓ found
- supabase-postgres: ✓ found

### Features
- Decision Log: ✓ enabled (docs/decisions/DECISIONS.md exists, ## Log anchor present)
- Flow Diagrams: ✓ enabled (mermaid format)
- Kanban: ✗ disabled

### Warnings
- ⚠ patterns.md is 45KB — consider compaction
- ⚠ 1 task stuck in_progress for 48h

### Overall: HEALTHY (2 warnings)
```
