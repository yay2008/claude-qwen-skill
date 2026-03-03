---
name: cc-pilot
description: Claude Code pilots coding tasks by analyzing requirements and designing solutions, then dispatching a partner AI (Qwen or Codex) to execute code changes via headless CLI. Use when user mentions Qwen or Codex in a coding context, such as "让Qwen做", "交给Qwen", "用Codex执行", "Codex帮我", "让Qwen改一下", "Qwen来写", "用Codex重构". Also triggers via /cc-pilot slash command. Supports --qwen and --codex backend flags for backend selection. Claude handles analysis, prompt construction, and code review; the chosen backend handles execution.
---

# CC Pilot: Claude Analyzes, Partner AI Executes

Claude Code collaborates with a partner AI (Qwen Code or OpenAI Codex) on coding tasks. Claude focuses on analysis and quality control, the partner AI focuses on code execution.

## Backend Selection

Parse the user's input to determine which backend to use:

- `/cc-pilot --qwen <task>` → use Qwen Code
- `/cc-pilot --codex <task>` → use OpenAI Codex
- `/cc-pilot <task>` (no flag) → default to Qwen

## Workflow

### Phase 1: Analyze

1. Read relevant code files using Glob/Grep/Read to understand context
2. Identify the technical stack, coding style, and comment language of the project
3. Design the solution approach
4. Construct a precise prompt for the partner AI following the template below

### Phase 2: Construct Prompt

Build a prompt using this template:

```
## Task
<one-line description of what to do>

## Context
- Project path: <cwd>
- Key files: <list of files to modify>
- Tech stack: <language/framework>

## Requirements
1. <specific step 1>
2. <specific step 2>
...

## Constraints
- <coding style / naming conventions>
- <files NOT to modify>
- <comment language must match existing code>
```

Prompt construction rules:
- Be specific, not abstract — say "rename function X to Y" not "improve naming"
- Scope modifications — list exact files to touch
- Include key context — paste relevant function signatures, types, data structures
- Set constraints — explicitly state what NOT to do

### Phase 3: Execute

Run the selected backend in headless mode:

**Qwen:**
```bash
qwen -p "<prompt>" --yolo 2>&1
```

**Codex:**
```bash
codex exec "<prompt>" --full-auto 2>&1
```

- Timeout: 120 seconds
- If the task description contains `--confirm`, show the constructed prompt to the user and wait for approval before executing

### Phase 4: Review

After execution completes, automatically review results:

1. Run `git diff` and `git status` to collect all changes
2. Review each changed file for:
   - **Correctness**: Does it fulfill the original requirement?
   - **Quality**: SOLID / KISS / DRY compliance
   - **Safety**: No obvious vulnerabilities or dangerous operations
   - **Completeness**: No missed requirements
3. Output a review report:

```
## Execution Review

**Backend:** Qwen / Codex
**Status:** PASS / WARN / FAIL

### Changes
- Modified: N files
- Added: N files
- Deleted: N files

### File Review
| File | Status | Notes |
|------|--------|-------|
| path/to/file | PASS/WARN/FAIL | explanation |

### Suggestions
- <specific fix suggestions if any>
```

## Error Handling

| Scenario | Action |
|----------|--------|
| Backend exits with non-zero code | Show error log, analyze cause, suggest fix |
| No files changed | Alert user, analyze why (prompt too vague, etc.) |
| Unintended files modified | Mark FAIL in review, suggest `git checkout` to revert |
| Execution timeout (>120s) | Kill process, report timeout |
| Backend CLI not found | Alert user, suggest installation |

## Important

- Claude MUST NOT write code directly — delegate all code changes to the partner AI
- Claude MUST review all changes after partner AI executes
- Never auto-commit — leave that decision to the user
