---
name: dev-workflow-router
user-invocable: true
description: |
  THE ENTRY POINT FOR ALL DEVELOPMENT TASKS. This skill MUST be activated for ANY development task.

  Use this skill when: building, implementing, debugging, fixing, reviewing, planning, refactoring, testing, or ANY coding request.

  Triggers: build, implement, create, make, write, add, develop, code, feature, component, screen, review, audit, check, analyze, debug, fix, error, bug, broken, troubleshoot, plan, design, architect, test, tdd, ui, backend, api, pattern, refactor, optimize, improve, enhance, update, modify, change.

  CRITICAL: Execute workflow immediately. Never just describe capabilities.
---

# Dev Workflow Router

**EXECUTION ENGINE.** Read config → Detect intent → Load memory → Load skills → Execute workflow → Update memory.

**NEVER** list capabilities. **ALWAYS** execute.

---

## HARD RULES (read first, always enforce)

1. **Every workflow MUST complete its full chain.** Do not skip agents unless skip_conditions match.
2. **Every agent MUST produce its required artifact** before the next agent starts.
3. **HANDOFF: ROUTER** from reviewer = workflow complete → run memory-update.
4. **Config is the source of truth** for constraints, test commands, and skip conditions.
5. **If Gemini MCP unavailable**, UI tasks fall back to builder-only (no error).

---

## Step 0: Load Project Config

```
Read(file_path=".claude/project.json")
```

Parse into `config`. If missing, use defaults:
```json
{
  "version": "1.0",
  "project": { "name": "project", "description": "", "tech_stack": [] },
  "agents": {
    "tester": { "test_command": "npm test" },
    "skip_conditions": {
      "tester": ["docs-only", "config-only", "user-says-skip"],
      "reviewer": ["docs-only", "config-only", "user-says-skip"],
      "designer": []
    }
  },
  "skills": {
    "project_patterns": null,
    "auto_load": {},
    "work_type_detection": {
      "ui": { "paths": ["screens/", "components/", "pages/"], "extensions": [".tsx", ".jsx"] },
      "backend": { "paths": ["services/", "api/", "server/", "db/"], "extensions": [".ts", ".js", ".sql"] },
      "domain": { "paths": ["domain/", "lib/", "utils/", "core/"] }
    }
  },
  "constraints": [],
  "integrations": {}
}
```

## Step 1: Detect Intent

| Priority | Signal | Keywords | Workflow |
|----------|--------|----------|----------|
| 1 | ERROR | error, bug, fix, broken, crash, fail, debug, troubleshoot, issue, problem, doesn't work | **DEBUG** |
| 2 | PLAN | plan, design, architect, roadmap, strategy, spec, "before we build", "how should we" | **PLAN** |
| 3 | REVIEW | review, audit, check, analyze, assess, "what do you think", "is this good" | **REVIEW** |
| 4 | DEFAULT | Everything else | **BUILD** |

**Conflict Resolution:** ERROR signals always win. "fix the build" = DEBUG (not BUILD).

## Step 2: Load Memory

**Step 2a - Create directory (sequential, before reads):**
```
Bash(command="mkdir -p .claude/memory")
```

**Step 2b - Load memory files (after mkdir completes):**
```
Read(file_path=".claude/memory/activeContext.md")
Read(file_path=".claude/memory/patterns.md")
Read(file_path=".claude/memory/progress.md")
```

If any file missing → create from `dev-workflow:memory` templates, then read.

## Step 3: Load Skills

### Always load project-patterns (if configured)
```
if config.skills.project_patterns:
  Read(file_path=config.skills.project_patterns)
```

### Detect work type and load matching skills
```
For each work_type in config.skills.work_type_detection:
  if user_request mentions matching paths/extensions:
    for skill_name in config.skills.auto_load[work_type]:
      Read(file_path=".claude/skills/{skill_name}/SKILL.md")
```

## Step 4: Evaluate Skip Conditions

Before creating the task hierarchy, check if any agents should be skipped:

```
skipped_agents = []

for agent_name, conditions in config.agents.skip_conditions:
  if "docs-only" in conditions AND all changed files are .md/.txt/.json:
    skipped_agents.append(agent_name)
  if "config-only" in conditions AND all changed files are config (no logic):
    skipped_agents.append(agent_name)
  if "user-says-skip" in conditions AND user explicitly said "skip tests" / "quick fix":
    skipped_agents.append(agent_name)
```

**When an agent is skipped, produce a SKIP_REPORT:**
```
SKIP_REPORT: {agent_name}
REASON: {matching condition}
SKIPPED_BY: router (config.agents.skip_conditions)
```

The SKIP_REPORT replaces that agent's required artifact for downstream validation.

## Agent Chains

| Workflow | Chain (full) | UI variant |
|----------|-------------|------------|
| BUILD | builder → tester → reviewer → memory-update | designer → builder → tester → reviewer → memory-update |
| DEBUG | builder → tester → reviewer → memory-update | same (no designer for debug) |
| REVIEW | reviewer → memory-update | same |
| PLAN | planner → codex-validate → memory-update | same |

**UI detection:** work type is `ui` AND Gemini MCP tools are available (`mcp__gemini__*`).
If Gemini MCP unavailable → use non-UI chain (builder starts).

## Task-Based Orchestration

### BUILD Workflow Tasks
```
# 1. Parent workflow task
TaskCreate({
  subject: "{project_name} BUILD: {feature_summary}",
  description: "Workflow: BUILD\nChain: {chain_description}",
  activeForm: "Building {feature}"
})

# 2a. Designer task (UI only, if not skipped)
if work_type == "ui" and gemini_available and "designer" not in skipped_agents:
  TaskCreate({ subject: "{project_name} designer: Generate UI for {feature}", activeForm: "Designing" })
  # Returns designer_task_id
  # builder_blocked_by = [designer_task_id]
else:
  # builder_blocked_by = []

# 2b. Builder task
TaskCreate({ subject: "{project_name} builder: Implement {feature}", activeForm: "Building" })
# Returns builder_task_id
# if designer: TaskUpdate({ taskId: builder_task_id, addBlockedBy: [designer_task_id] })

# 2c. Tester task (if not skipped)
if "tester" not in skipped_agents:
  TaskCreate({ subject: "{project_name} tester: Test implementation", activeForm: "Testing" })
  TaskUpdate({ taskId: tester_task_id, addBlockedBy: [builder_task_id] })
  reviewer_blocked_by = tester_task_id
else:
  # Produce SKIP_REPORT for tester
  reviewer_blocked_by = builder_task_id

# 2d. Reviewer task (if not skipped)
if "reviewer" not in skipped_agents:
  TaskCreate({ subject: "{project_name} reviewer: Review changes", activeForm: "Reviewing" })
  TaskUpdate({ taskId: reviewer_task_id, addBlockedBy: [reviewer_blocked_by] })
  memory_blocked_by = reviewer_task_id
else:
  # Produce SKIP_REPORT for reviewer
  memory_blocked_by = reviewer_blocked_by

# 3. Memory Update task
TaskCreate({
  subject: "{project_name} Memory Update: Persist learnings",
  description: "Collect Memory Notes from agents, persist to .claude/memory/ files.\nUse Read-Edit-Read pattern.",
  activeForm: "Persisting learnings"
})
TaskUpdate({ taskId: memory_task_id, addBlockedBy: [memory_blocked_by] })
```

### PLAN Workflow Tasks
```
TaskCreate({ subject: "{project_name} PLAN: {feature}", activeForm: "Planning" })

TaskCreate({ subject: "{project_name} planner: Create plan", activeForm: "Creating plan" })
# Returns planner_task_id

TaskCreate({
  subject: "{project_name} codex-validate: Review plan",
  description: "Spawn Codex for dry-run validation. Max 3 iterations.",
  activeForm: "Validating plan"
})
TaskUpdate({ taskId: codex_task_id, addBlockedBy: [planner_task_id] })

TaskCreate({ subject: "{project_name} Memory Update: Index plan", activeForm: "Indexing plan" })
TaskUpdate({ taskId: memory_task_id, addBlockedBy: [codex_task_id] })
```

### REVIEW Workflow Tasks
```
TaskCreate({ subject: "{project_name} REVIEW: {scope}", activeForm: "Reviewing" })

TaskCreate({ subject: "{project_name} reviewer: Review {scope}", activeForm: "Reviewing" })

TaskCreate({ subject: "{project_name} Memory Update: Persist findings", activeForm: "Persisting" })
TaskUpdate({ taskId: memory_task_id, addBlockedBy: [reviewer_task_id] })
```

## Agent Invocation

**Pass task ID, config, and context to each agent:**

```
Agent(
  subagent_type="{agent_name}",
  prompt="
## Task Context
- **Task ID:** {taskId}
- **Project:** {config.project.name}

## User Request
{request}

## Project Constraints
{config.constraints joined by newlines}

## Test Command
{config.agents.tester.test_command}

## Tech Stack
{config.project.tech_stack joined by comma}

## Memory Summary
{brief from activeContext.md}

## Project Patterns
{key from patterns.md}

## Skills Loaded
{skills list}

---
Execute the task. Include 'Task {TASK_ID}: COMPLETED' when done.
"
)
```

## Post-Agent Validation

| Agent | Required Artifact | Skippable? |
|-------|-------------------|------------|
| designer | HANDOFF: BUILDER + GENERATED_CODE | Yes (if no Gemini MCP) |
| builder | HANDOFF: TESTER + CHANGES + ACCEPTANCE | No |
| tester | TEST_REPORT: PASS/FAIL/MANUAL | Yes (via skip_conditions) |
| reviewer | REVIEW_REPORT + FINAL_STATUS | Yes (via skip_conditions) |
| planner | Specification + Task breakdown + Confidence | No |
| codex | CODEX_REVIEW: APPROVED/ISSUES_FOUND | No |

**If agent was skipped:** SKIP_REPORT replaces its artifact. Downstream agents see SKIP_REPORT instead.

**If required artifact missing (non-skipped agent):**
```
TaskCreate({
  subject: "{project_name} REMEDIATION: {agent} missing {artifact}",
  description: "Re-run agent to produce required output.",
  activeForm: "Collecting missing evidence"
})
TaskUpdate({ taskId: downstream_task_id, addBlockedBy: [remediation_task_id] })
```

**STOP. Do not invoke next agent until validation passes.**

## Codex Validation (PLAN only)

1. Extract plan path from planner output (`### Plan Location`)
2. Read plan content
3. Spawn Codex:
```
mcp__codex-subagent__spawn_agent({
  prompt: "Review this plan for a {tech_stack} project.
  RULES: Read-only. No file modifications.
  CONSTRAINTS: {constraints}
  PLAN: {plan_content}
  CHECK: files exist? compatible with structure? missing deps? architectural conflicts? realistic tasks?
  OUTPUT: CODEX_REVIEW: APPROVED|ISSUES_FOUND, CONFIDENCE: X/10, ISSUES: (if any)"
})
```
4. If APPROVED → memory-update
5. If ISSUES_FOUND → re-invoke planner with `CODEX_FEEDBACK:` (max 3 iterations)
6. After 3 failures → warn and proceed

## Integration Sync (Config-Driven)

After memory update, for each integration in `config.integrations`:
- If `enabled: true` → run integration-specific sync
- Integration logic is project-specific (ClickUp, Linear, Jira, etc.)

## Workflow Execution Summary

### BUILD
1. Load config → Load memory → Check progress.md
2. Detect work type → Load skills → Evaluate skip conditions
3. Clarify if ambiguous (AskUserQuestion)
4. Create task hierarchy (with/without designer, with/without tester/reviewer)
5. Execute chain, validate each agent output
6. Integration sync → Memory update

### DEBUG
1. Load config → Load memory → Check patterns.md gotchas
2. Clarify if ambiguous
3. Create task hierarchy (no designer)
4. Execute chain with builder in fix mode
5. Memory update → Add to Common Gotchas

### REVIEW
1. Load config → Load memory → Confirm scope
2. Execute reviewer → Memory update

### PLAN
1. Load config → Load memory
2. Execute planner → Codex loop → Memory update

## Error Recovery

1. Check which gate failed
2. Identify missing handoff/report
3. Create remediation task
4. Route back to appropriate agent
5. Do NOT proceed without required artifacts
