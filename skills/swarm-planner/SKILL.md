---
name: swarm-planner
description: >
  [EXPLICIT INVOCATION ONLY] Creates dependency-aware implementation plans optimized for parallel
  multi-agent execution.
metadata:
  invocation: explicit-only
---

# Swarm-Ready Planner

Create implementation plans with explicit task dependencies optimized for parallel agent execution. This skill can be ran inside or outside of Plan Mode.

## Core Principles

1. **Explore Codebase**: Investigate architecture, patterns, existing implementations, dependencies, and frameworks in use.
2. **Fresh Documentation First**: Use Context7 for ANY external library, framework, or API before planning tasks
3. **Ask Questions**: Clarify ambiguities and seek clarification on scope, constraints, or priorities throughout the planning process. At any time.
4. **Explicit Dependencies**: Every task declares what it depends on, enabling maximum parallelization
5. **Atomic Tasks**: Each task is independently executable by a single agent
6. **Review Before Yield**: A subagent reviews the plan for gaps before finalizing

## Output Structure

Plans are saved into an organized directory structure:

```
plans/
  YYYY-MM-DD-HH-MM-topic/
    plan.md              ← master index (task summary + plan_file references)
    phase-01-[summary].md ← full detailed plan (where execution happens)
```

- **Directory name**: `plans/YYYY-MM-DD-HH-MM-[topic-slug]` (timestamp + kebab-case topic)
- **`plan.md`**: Master index with task summaries, `plan_file` paths, and execution metadata
- **`phase-01-[summary].md`**: Full detailed plan with complete task descriptions, acceptance criteria, and validation steps

## Process

### 1. Research

**Codebase investigation:**
- Architecture, patterns, existing implementations
- Dependencies and frameworks in use

### 1a. Optional: Stop to Clarification Questions

- If the architecture is unclear or missing STOP AND YIELD to the user, and request user input (AskUserQuestions) before moving on. Always offer recommendations for clarification questions.
- If architecture is present, skip 1a and move onto next step.

### 2. Documentation

**Documentation retrieval (REQUIRED for external dependencies):**

Use Context7 skill or MCP to fetch current docs for any libraries/frameworks or APIs that are or will be used in project. If Context7 is not available, use web search.

This ensures version-accurate APIs, correct parameters, and current best practices.

### 3. STOP and Request User Input

When anything is unclear or could reasonably be done multiple ways:
- Stop and ask clarifying questions immediately
- Do not make assumptions about scope, constraints, or priorities
- Questions should reduce risk and eliminate ambiguity
- Always offer recommendations for clarification questions.
- Use request_user_input or AskUserQuestion tool if available.

### 4. Create Directory and Write Plan Files

Create the timestamped directory: `plans/YYYY-MM-DD-HH-MM-[topic-slug]/`

**`YYYY-MM-DD-HH-MM`** = current datetime in UTC (24h format)
**`topic-slug`** = kebab-case summary of the topic (e.g., `auth`, `user-dashboard`, `payment-integration`)

#### Write `phase-01-[summary].md` (Detailed Plan)

This is the **authoritative** plan file. Execution skills read this file for full task details. Include:

```markdown
# Plan: [Full Topic Name]

**Generated**: YYYY-MM-DD HH:MM UTC
**Topic**: [topic-slug]

## Overview
[Summary of task and approach]

## Prerequisites
- [Tools, libraries, access needed]

## Dependency Graph

```
T1 ──┬── T3 ──┐
     │        ├── T5 ── T6 ── T7
T2 ──┴── T4 ──┘
```

## Tasks

### T1: [Name]
- **depends_on**: []
- **location**: [file paths]
- **description**: [detailed what to do]
- **acceptance_criteria**:
  - [Criterion 1]
  - [Criterion 2]
- **validation**: [how to verify — exact commands or steps]
- **status**: Not Completed
- **log**: [leave empty, to be filled out later]
- **files edited/created**: [leave empty, to be filled out later]

### T2: [Name]
- **depends_on**: []
- **location**: [file paths]
- **description**: [detailed what to do]
- **acceptance_criteria**:
  - [Criterion 1]
- **validation**: [how to verify]
- **status**: Not Completed
- **log**: [leave empty, to be filled out later]
- **files edited/created**: [leave empty, to be filled out later]

[... continue for all tasks ...]

## Parallel Execution Groups

| Wave | Tasks | Can Start When |
|------|-------|----------------|
| 1 | T1, T2 | Immediately |
| 2 | T3, T4 | Wave 1 complete |
| 3 | T5 | T3, T4 complete |

## Testing Strategy
- [How to test]
- [What to verify]

## Risks & Mitigations
- [What could go wrong + how to handle]
```

#### Write `plan.md` (Master Index)

This is the **entry point** for execution skills. Include:

```markdown
# Plan Index: [Topic Name]

**Generated**: YYYY-MM-DD HH:MM UTC
**Plan Directory**: plans/YYYY-MM-DD-HH-MM-[topic-slug]/

## plan_file Reference

Full detailed plan: `plans/YYYY-MM-DD-HH-MM-[topic-slug]/phase-01-[summary].md`

Execution skills should read the `plan_file` for complete task details.

## Task Summary

| ID | Name | Depends On | Location | Status |
|----|------|-----------|----------|--------|
| T1 | [Name] | [] | [files] | Not Completed |
| T2 | [Name] | [] | [files] | Not Completed |
| T3 | [Name] | [T1] | [files] | Not Completed |

## Parallel Execution Groups

| Wave | Tasks | Can Start When |
|------|-------|----------------|
| 1 | T1, T2 | Immediately |
| 2 | T3 | Wave 1 complete |

## Execution Commands

To run: `/parallel-task plans/YYYY-MM-DD-HH-MM-[topic-slug]/plan.md`
```

### 5. Subagent Review

After saving both files, spawn a subagent to review the plan:

```
Review this implementation plan for:
1. Missing dependencies between tasks
2. Ordering issues that would cause failures
3. Missing error handling or edge cases
4. Gaps, holes, gotchas.

Provide specific, actionable feedback. Do not ask questions.

Plan directory: [directory path]
Plan index: [plan.md path]
Detailed plan: [phase-01-[summary].md path]
Context: [brief context about the task]
```

If the subagent provides actionable feedback, revise the plan before yielding.


## Task Dependency Format

Each task MUST include:
- **id**: Unique identifier (e.g., `T1`, `T2.1`)
- **depends_on**: Array of task IDs that must complete first (empty `[]` for root tasks)
- **description**: What the task accomplishes
- **location**: File paths involved
- **validation**: acceptance criteria

**Example:**
```
T1: [depends_on: []] Create database schema migration
T2: [depends_on: []] Install required packages
T3: [depends_on: [T1]] Create repository layer
T4: [depends_on: [T1]] Create service interfaces
T5: [depends_on: [T3, T4]] Implement business logic
T6: [depends_on: [T2, T5]] Add API endpoints
T7: [depends_on: [T6]] Write integration tests
```

Tasks with empty/satisfied dependencies can run in parallel (T1, T2 above).

## Important

- Every task must have explicit `depends_on` field
- Root tasks (no dependencies) can be executed in parallel immediately
- Do NOT implement - only create the plan
- Always use Context7 for external dependencies before finalizing tasks
- Always ask questions where ambiguity exists
- Save to the timestamped directory structure, NOT a single file in CWD
- The `plan_file` in `plan.md` points to the detailed phase plan for execution
