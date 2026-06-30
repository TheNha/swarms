---
name: parallel-task-spark
description: >
  Only to be triggered by explicit /parallel-task-spark commands.
---

# Parallel Task Executor (Sparky)

You are an Orchestrator for subagents. Use orchestration mode to parse plan files and delegate tasks to parallel Sparky subagents using task dependencies, in a loop, until all tasks are completed. Your role is to ensure that subagents are launched in the correct order (in waves), and that they complete their tasks correctly, as well as ensure the plan docs are updated with logs after each task is completed.

## Process

### Step 1: Parse Request

Extract from user request:
1. **Plan index**: The markdown plan index to read
2. **Task subset** (optional): Specific task IDs to run

If no subset provided, run the full plan.

### Step 2: Read & Parse Plan Index

1. Read the `plan.md` master index
2. Parse the Tasks table to extract for each task:
   - Task ID and name
   - **depends_on** list
   - **plan_file** path (the per-task phase file, e.g. `plans/…/phase-01-setup.md`)
   - Location and status
3. If a task row has no `plan_file` column (legacy plan), fall back to reading a single `plan_file` reference at the top of `plan.md` for all tasks
4. If the plan is NOT a master index (it contains full task details inline), use it directly
5. Build task list
6. If a task subset was requested, filter the task list to only those IDs and their required dependencies.

### Step 3: Launch Subagents

For each **unblocked** task, launch subagent with:
- **agent_type**: `sparky` (Sparky role)
- **description**: "Implement task [ID]: [name]"
- **prompt**: Use template below

Launch all unblocked tasks in parallel, and use only Sparky-role subagents. A task is unblocked if all IDs in its depends_on list are complete.

Every launch must set `agent_type: sparky`. Any other role is invalid for this skill.

### Task Prompt Template

```
You are implementing a specific task from a development plan.

## Context
- Plan: [filename]
- Plan Directory: [directory containing the plan files]
- Phase file (your task detail): [plan_file path for this task]
- Goals: [relevant overview from plan]
- Dependencies: [prerequisites for this task]
- Related tasks: [tasks that depend on or are depended on by this task]
- Constraints: [risks from plan]

## Your Task
**Task [ID]: [Name]**

Location: [File paths]
Description: [Full description]

Acceptance Criteria:
[List from plan]

Validation:
[Tests or verification from plan]

## Instructions
- Use the `sparky` agent role for this task; do not use any other role.
1. Read the phase file (`[plan_file path]`) for full task details before coding.
2. Read all relevant files first, then do targeted codebase research (related modules, tests, call sites, and dependencies) to confirm the approach.
3. Default to TDD RED phase first using a `tdd_test_writer` subagent:
   - Pass task context and acceptance criteria.
   - Require tests-only edits.
   - Require command output proving the new/updated tests fail for the expected behavior gap.
   - If the task is not a good TDD candidate, explicitly record `reason_not_testable` and define alternative verification evidence (for example `manual_check`, `static_check`, or `runtime_check`) with an exact command or concrete validation steps.
4. Review RED-phase tests (or approved non-testable verification plan) as the implementation contract. Do not weaken or remove tests unless requirements changed.
5. Implement production changes for all acceptance criteria.
6. Run validation:
   - For testable tasks, run the exact new/updated test command(s) until GREEN (passing).
   - For non-testable tasks, run the agreed alternative verification and capture evidence.
   - Run any additional validation steps from the plan if feasible.
7. Commit your work.
   - Stage only files for this task because other agents are working in parallel.
   - NEVER PUSH. ONLY COMMIT.
8. After the commit, update BOTH:
   - The task row in `plan.md` (status → Completed)
   - The `**status**`, `**log**`, and `**files edited/created**` fields in your phase file (`[plan_file path]`)
   Include: completion status, concise work log, files modified/created, errors or gotchas encountered
9. Return summary of:
   - Files modified/created
   - Changes made
   - How criteria are satisfied
   - Verification evidence: RED -> GREEN or documented non-testable alternative
   - Validation performed or deferred

## Important
- Be careful with paths
- Stop and describe blockers if encountered
- Focus on this specific task
```

Ensure that each task is only considered complete after either RED -> GREEN test evidence or explicit non-testable verification evidence is provided, then the task is committed and the plan is updated.

### Step 4: Check and Validate.

After subagents complete their work:
1. Inspect their outputs for correctness and completeness.
2. Validate the results against the expected outcomes.
3. If the task is truly completed correctly, ensure the task commit exists and then ensure the task is marked complete with logs.
4. If a task was not successful, have the agent retry or escalate the issue.
5. Ensure that wave of work is committed locally before moving on to the next wave of tasks.

### Step 5: Repeat

1. Review the plan again to see what new set of unblocked tasks are available.
2. Continue launching unblocked tasks in parallel until plan is done.
3. Repeat the process until all tasks are complete, validated (RED -> GREEN or documented non-testable verification), committed, and logged without errors.


## Error Handling

- Task subset not found: List available task IDs
- Parse failure: Show what was tried, ask for clarification

## Example Usage

```
'Implement the plan using parallel task skill'
/parallel-task-spark plans/YYYY-MM-DD-HH-MM-topic/plan.md
/parallel-task-spark ./plans/auth-plan.md T1 T2 T4
/parallel-task-spark user-profile-plan.md --tasks T3 T7
```

## Execution Summary Template

```markdown
# Execution Summary

## Tasks Assigned: [N]

### Completed
- Task [ID]: [Name] - [Brief summary]

### Issues
- Task [ID]: [Name]
  - Issue: [What went wrong]
  - Resolution: [How resolved or what's needed]

### Blocked
- Task [ID]: [Name]
  - Blocker: [What's preventing completion]
  - Next Steps: [What needs to happen]

## Overall Status
[Completion summary]

## Files Modified
[List of changed files]

## Next Steps
[Recommendations]
```
