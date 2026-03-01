# /eng-team Plugin — Workflow Reference

This document describes the full workflow, phases, gates, label lifecycle, and agent communication patterns for the `/eng-team` plugin. Agents can reference this document when they need to understand where they fit in the overall process.

---

## Overview

The `/eng-team` command orchestrates a full engineering team through four sequential phases, each separated by a user approval gate. All work is tracked in Linear.

```
Input → Phase 1: Product Discovery → GATE
      → Phase 2: Architecture       → GATE
      → Phase 3: Engineering Design → GATE
      → Phase 4: Implementation     → Done
```

---

## Agents and Roles

| Agent | Model | Phase(s) | Responsibility |
|-------|-------|----------|----------------|
| Lead (Opus) | opus | All | Orchestration, gates, user communication |
| ProductMgr | opus | 1 | User stories, acceptance criteria, Linear project |
| SWA | opus | 2 | Architecture planning, schema_change ticket |
| SWEL | opus | 2 + 3 | Architecture validation, engineering design |
| EM | sonnet | 4 | Implementation coordination, assignment, progress |
| BEEng | sonnet | 4 | Backend implementation (TDD) |
| FEEng | sonnet | 4 | Frontend implementation (TDD) |
| E2EEng | sonnet | 3 | E2E test planning (tickets only, no code) |

---

## Linear Label Lifecycle

Labels are created by the Lead in Step 3 before any agents are spawned.

```
ProductMgr creates ticket
  → labels: ["ready_for_eng_design"]

SWEL picks up ticket (Phase 3)
  → adds engineering detail
  → removes: "ready_for_eng_design"

  If ticket needs BOTH backend + frontend:
    → creates subissue labeled: ["backend", "ready_for_implementation"]
    → creates subissue labeled: ["frontend", "ready_for_implementation"]
    → parent ticket: no layer label (it's a container)

  If ticket needs ONLY one layer:
    → labels parent: ["backend" OR "frontend", "ready_for_implementation"]

Special tickets:
  SWA creates: Architecture ticket (no special label)
  SWA creates: schema_change ticket → labels: ["schema_change"], priority: 1 (Urgent)
  E2EEng creates: E2E tickets → labels: ["e2e"]

EM monitors: tickets labeled "ready_for_implementation"
EM assigns tickets → engineer implements → ticket moves to Done state
```

---

## Phase Details

### Phase 1: Product Discovery

**Agents active:** ProductMgr

**What happens:**
1. ProductMgr creates a Linear project for the feature
2. ProductMgr asks the user clarifying questions (one at a time) about personas, outcomes, and scope
3. ProductMgr creates user story tickets (one per distinct story)
4. Each ticket has: user story format, acceptance criteria, TDD requirements
5. All tickets labeled: `ready_for_eng_design`

**Gate:** Lead presents user story titles to user and asks for approval. User can request revisions.

**Output:**
- Linear project (with projectId)
- N user story tickets, all labeled `ready_for_eng_design`

---

### Phase 2: Architecture

**Agents active:** SWA + SWEL (both active simultaneously, communicating via SendMessage)

**What happens:**
1. SWA spawns exploration subagents to understand the existing codebase
2. SWA drafts an architecture covering: data model, API, frontend, infrastructure, integration points
3. SWA sends draft to SWEL via SendMessage
4. SWEL spawns subagents to validate the architecture against codebase reality
5. SWEL sends feedback/corrections to SWA
6. Iteration until both agree
7. SWA creates the Architecture ticket in Linear
8. If schema changes needed: SWA creates the schema_change ticket (priority 1, blocks all implementation)
9. Both agents send completion messages to Lead

**Gate:** Lead presents architecture ticket and schema_change ticket (if any) to user. User can request revisions.

**Output:**
- Architecture ticket (documents the technical plan)
- schema_change ticket (if needed) — highest priority, blocks all implementation

---

### Phase 3: Engineering Design

**Agents active:** SWEL (continues from Phase 2), optionally E2EEng

**What happens:**
1. SWEL fetches all `ready_for_eng_design` tickets from Linear
2. For each ticket, SWEL:
   a. Spawns a subagent to explore the specific code areas the ticket touches
   b. Updates the ticket description with full engineering detail (files, function signatures, test cases)
   c. Decides: backend only, frontend only, or both
   d. If both: creates backend + frontend subissues (both labeled `ready_for_implementation`)
   e. If one layer: labels the parent ticket and marks `ready_for_implementation`
   f. Sets blocking relationships between tickets
3. SWEL assesses whether E2E tests are warranted; reports to Lead
4. Lead optionally spawns E2EEng agent
5. E2EEng reads user stories and creates E2E test scenario tickets (labeled `e2e`)

**Gate:** Lead presents all `ready_for_implementation` tickets and dependency order to user. User can request revisions.

**Output:**
- All user story tickets refined with engineering detail
- Backend/frontend subissues created where needed
- Dependency order established (blocking relationships set)
- E2E test tickets created (if applicable)

---

### Phase 4: Implementation

**Agents active:** EM + BEEng(s) + FEEng(s)

**What happens:**
1. EM fetches all `ready_for_implementation` tickets
2. If schema_change ticket exists:
   - EM notifies Lead → Lead notifies user to run migrations
   - Wait for user confirmation
   - Mark schema_change ticket Done
3. EM assigns backend tickets to BEEng, frontend tickets to FEEng
4. Engineers follow Red/Green TDD:
   - Write failing unit tests
   - Write failing integration tests (or component tests)
   - Implement minimum code to pass each test
   - Refactor
   - Full suite passes
5. Engineers mark tickets Done in Linear and report to EM
6. EM unblocks dependent tickets and assigns them
7. EM escalates blockers to SWA/SWEL/ProductMgr as needed
8. EM reports all-done to Lead

**Output:**
- All tickets marked Done in Linear
- Feature implemented with full test coverage

---

## Agent Communication Patterns

```
Lead ──SendMessage──> ProductMgr      (spawn + instructions)
ProductMgr ──SendMessage──> Lead      (completion report)

Lead ──SendMessage──> SWA             (spawn + instructions)
Lead ──SendMessage──> SWEL            (spawn + instructions)
SWA ──SendMessage──> SWEL             (architecture draft, questions)
SWEL ──SendMessage──> SWA             (feedback, validation results)
SWA ──SendMessage──> Lead             (Phase 2 completion)
SWEL ──SendMessage──> Lead            (Phase 2 completion)

Lead ──SendMessage──> SWEL            (Phase 3 begin signal)
SWEL ──SendMessage──> Lead            (Phase 3 completion + E2E assessment)

Lead ──SendMessage──> E2EEng          (spawn + instructions, if warranted)
E2EEng ──SendMessage──> Lead          (completion report)

Lead ──SendMessage──> EM              (spawn + instructions)
Lead ──SendMessage──> BEEng(s)        (spawn + instructions)
Lead ──SendMessage──> FEEng(s)        (spawn + instructions)
EM ──SendMessage──> BEEng             (ticket assignment)
EM ──SendMessage──> FEEng             (ticket assignment)
BEEng ──SendMessage──> EM             (ticket completion)
FEEng ──SendMessage──> EM             (ticket completion)
EM ──SendMessage──> Lead              (migration needed, status updates, completion)
EM ──SendMessage──> SWA               (technical blocker escalation)
EM ──SendMessage──> SWEL              (implementation question escalation)
```

### Message Acknowledgment Protocol

When an agent receives a revision or high-priority message while working on other tasks, it MUST:
1. Reply immediately with a brief acknowledgment: "Received [summary of message]. Processing now."
2. Prioritize the revision over current work
3. Send a completion message when the revision is applied

The lead MUST NOT send multiple messages to the same agent in rapid succession. Instead, batch related instructions into a single message. If a follow-up is truly urgent, prefix the message with `[PRIORITY]` to signal it overrides current work.

---

## Key Constraints and Rules

### For All Agents
- **Linear MCP tools**: Use `mcp__claude_ai_Linear__*` for all Linear operations
- **Figma MCP tools**: Use `mcp__plugin_figma_figma__*` for design context
- **No Serena in worktrees**: Use native Read/Edit/Glob/Grep tools in implementation contexts
- **Send messages, not text output**: Agents must use SendMessage to communicate — plain text output is not visible to other agents
- **Ticket invalidation**: When an architecture revision invalidates an existing ticket, the responsible agent must either delete or update it and confirm the action in their completion message

### For Opus Agents (Lead, ProductMgr, SWA, SWEL)
- Spawn subagents for exploration — don't explore the codebase yourself
- Protect your context window by delegating
- Prefer Haiku subagents for pure exploration tasks; Sonnet for tasks requiring synthesis

### For Sonnet Agents (EM, BEEng, FEEng, E2EEng)
- Do not spawn additional agents — you are the implementation layer
- Follow Red/Green TDD without exception
- Read the project's CLAUDE.md before implementing
- Use native tools (Read/Glob/Grep/Edit/Write) for code operations

### For the Lead Specifically
- Create labels before spawning agents (they need label IDs)
- Pass projectId to every agent once ProductMgr creates it
- Pass labelIds (map) to every agent so they don't re-fetch
- Surface schema_change tickets to the user for migration instruction
- Collect completion messages from all agents before proceeding to next phase

---

## TDD Red/Green/Refactor Summary

All implementation agents follow this strict cycle:

```
RED:
  Write failing unit test → run → confirm it fails
  Write failing integration/component test → run → confirm it fails

GREEN (unit tests):
  Write minimum implementation to pass unit test → run → confirm green
  Repeat for each unit test

GREEN (integration tests):
  Run integration tests once unit tests pass and implementation is complete
  Fix failures until all integration tests pass

REFACTOR:
  Clean up code while keeping all tests green
  Run linter → fix any issues

FULL SUITE:
  Run all tests (pnpm test or pnpm test:frontend)
  Confirm no regressions before marking ticket done
```

---

## Linear Tool Reference

| Operation | Tool |
|-----------|------|
| Create/update project | `mcp__claude_ai_Linear__save_project` |
| Create/update issue | `mcp__claude_ai_Linear__save_issue` |
| Get issue details | `mcp__claude_ai_Linear__get_issue` |
| List issues (with filters) | `mcp__claude_ai_Linear__list_issues` |
| Create label | `mcp__claude_ai_Linear__create_issue_label` |
| List labels | `mcp__claude_ai_Linear__list_issue_labels` |
| Add comment | `mcp__claude_ai_Linear__create_comment` |
| List teams | `mcp__claude_ai_Linear__list_teams` |
| List issue statuses | `mcp__claude_ai_Linear__list_issue_statuses` |
| List projects | `mcp__claude_ai_Linear__list_projects` |
