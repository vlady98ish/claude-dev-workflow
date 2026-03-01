---
name: dev-workflow-memory
user-invocable: true
description: "Internal skill for memory management. Provides Read-Edit-Verify pattern, stable anchors, and pre-compaction checkpoint."
allowed-tools: Read, Write, Edit, Bash
---

# Session Memory (MANDATORY)

## The Iron Law

```
EVERY WORKFLOW MUST:
1. LOAD memory at START (and before key decisions)
2. UPDATE memory at END (and after learnings/decisions)
```

**Brevity Rule:** Memory is an index, not a document. Be brief—one line per item.

## Memory Structure

```
.claude/memory/
├── activeContext.md   # Current focus + learnings + decisions (MOST IMPORTANT)
├── patterns.md        # Project patterns, conventions, gotchas
└── progress.md        # What works, what's left, verification evidence
```

## Permission-Free Operations (CRITICAL)

**ALL memory operations are PERMISSION-FREE using the correct tools.**

| Operation | Tool | Permission |
|-----------|------|------------|
| Create memory directory | `Bash(command="mkdir -p .claude/memory")` | FREE |
| **Read memory files** | `Read(file_path=".claude/memory/activeContext.md")` | **FREE** |
| **Create NEW memory file** | `Write(file_path="...", content="...")` | **FREE** (file doesn't exist) |
| **Update EXISTING memory** | `Edit(file_path="...", old_string="...", new_string="...")` | **FREE** |

### CRITICAL: Write vs Edit

| Tool | Use For | Asks Permission? |
|------|---------|------------------|
| **Write** | Creating NEW files | NO (if file doesn't exist) |
| **Write** | Overwriting existing files | **YES** |
| **Edit** | Updating existing files | **NO - always permission-free** |

**RULE: Use Write for NEW files, Edit for UPDATES.**

## Stable Anchors (ONLY use these)

| Anchor | File | Stability |
|--------|------|-----------|
| `## Current Focus` | activeContext | GUARANTEED |
| `## Recent Changes` | activeContext | GUARANTEED |
| `## Learnings` | activeContext | GUARANTEED |
| `## Decisions` | activeContext | GUARANTEED |
| `## References` | activeContext | GUARANTEED |
| `## Last Updated` | all files | GUARANTEED (fallback) |
| `## Common Gotchas` | patterns | GUARANTEED |
| `## Architecture Patterns` | patterns | GUARANTEED |
| `## Tasks` | progress | GUARANTEED |
| `## Completed` | progress | GUARANTEED |
| `## Verification` | progress | GUARANTEED |

**NEVER use as anchors:**
- Table headers
- Checkbox text
- Optional sections that may not exist

## Read-Edit-Verify (MANDATORY)

Every memory edit MUST follow this exact sequence:

### Step 1: READ
```
Read(file_path=".claude/memory/activeContext.md")
```

### Step 2: VERIFY ANCHOR
```
# Check if intended anchor exists in the content you just read
# If "## References" not found → use "## Last Updated" as fallback
```

### Step 3: EDIT
```
Edit(file_path=".claude/memory/activeContext.md",
     old_string="## Recent Changes",
     new_string="## Recent Changes\n- [New entry]\n")
```

### Step 4: VERIFY
```
Read(file_path=".claude/memory/activeContext.md")
# Confirm your change appears. If not → STOP and retry.
```

## Pre-Compaction Safety (CRITICAL)

**Update memory IMMEDIATELY when you notice:**
- Extended debugging (5+ cycles)
- Long planning discussions
- Multi-file refactoring
- 30+ tool calls in session

**Checkpoint Pattern:**
```
Edit(file_path=".claude/memory/activeContext.md",
     old_string="## Current Focus",
     new_string="## Current Focus\n\n[Updated focus + key decisions]")
Read(file_path=".claude/memory/activeContext.md")  # Verify
```

**Rule:** When in doubt, update memory NOW. Better duplicate entries than lost context.

## Memory File Templates

### activeContext.md
```markdown
# Active Context

## Current Focus
[Active work]

## Recent Changes
- [Change] - [file:line]

## Next Steps
1. [Step]

## Decisions
- [Decision]: [Choice] - [Why]

## Learnings
- [Insight]

## References
- Plan: N/A
- Design: N/A
- Research: N/A

## Blockers
- [None]

## Last Updated
[timestamp]
```

### patterns.md
```markdown
# Project Patterns

## Architecture Patterns
- [Pattern]: [How this project implements it]

## Code Conventions
- [Convention]: [Example]

## Common Gotchas
- [Gotcha]: [How to avoid / solution]

## Testing Patterns
- [Test type]: [How to write, where to put]

## Dependencies
- [Dependency]: [Why used, how configured]

## Last Updated
[timestamp]
```

### progress.md
```markdown
# Progress Tracking

## Current Workflow
[PLAN | BUILD | REVIEW | DEBUG]

## Tasks
- [ ] Task 1
- [x] Task 2 - evidence

## Completed
- [x] Item - evidence

## Verification
- `command` → exit 0 (X/X)

## Last Updated
[timestamp]
```

## Mandatory Operations

### At Workflow START (REQUIRED)

```
# Step 1: Create directory
Bash(command="mkdir -p .claude/memory")

# Step 2: Load ALL 3 memory files
Read(file_path=".claude/memory/activeContext.md")
Read(file_path=".claude/memory/patterns.md")
Read(file_path=".claude/memory/progress.md")
```

### At Workflow END (REQUIRED)

```
# 1. Update memory files (Read-Edit-Verify)
Read(file_path=".claude/memory/activeContext.md")

Edit(file_path=".claude/memory/activeContext.md",
     old_string="## Recent Changes",
     new_string="## Recent Changes\n- [YYYY-MM-DD] [What changed] - [file:line]\n")

Read(file_path=".claude/memory/activeContext.md")  # VERIFY
```

## Memory Notes Section (For READ-ONLY Agents)

If an agent cannot safely update memory (no `Edit` tool):

Include in output:
```markdown
### Memory Notes (For Workflow-Final Persistence)
- **Learnings:** [insights for activeContext.md]
- **Patterns:** [gotchas for patterns.md]
- **Verification:** [results for progress.md]
```

Router will persist these notes via the Memory Update task.

## Red Flags - STOP IMMEDIATELY

If you catch yourself:
- Starting work WITHOUT loading memory
- Making decisions WITHOUT checking Decisions section
- Completing work WITHOUT updating memory
- Saying "I'll remember" instead of writing to memory

**STOP. Load/update memory FIRST.**
