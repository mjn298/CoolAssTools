# CoolAssTools

Shared workflows and tools for [Claude Code](https://docs.anthropic.com/en/docs/claude-code).

## What is this?

This repo contains slash commands that automate product and engineering workflows. You type a command in Claude Code and it handles the rest — asking clarifying questions, creating tickets, coordinating agents.

| Command | What it does |
|---------|-------------|
| `/refine-tickets` | Takes rough feature descriptions and turns them into well-structured Asana tickets |
| `/eng-team` | Spins up a full AI engineering team to plan and build a feature |

## Prerequisites

1. **Claude Code** — Install from [claude.ai/code](https://claude.ai/code) or via `npm install -g @anthropic-ai/claude-code`
2. **MCP Plugins** — These commands use external services via MCP (Model Context Protocol):
   - **Asana MCP** — Required for `/refine-tickets`. Install via Claude Code's plugin marketplace (`/plugins` → search "asana")
   - **Linear MCP** — Required for `/eng-team`. Install via plugin marketplace (`/plugins` → search "linear")
   - **Figma MCP** — Optional, for `/eng-team` when providing Figma design URLs

## Installation

### Add the marketplace (recommended)

This is the easiest way. It adds the CoolAssTools marketplace to Claude Code so you can install and update plugins from it.

1. Open Claude Code
2. Run `/plugins`
3. Choose "Add marketplace"
4. Enter: `CoolAssTools/CoolAssTools`
5. Install `cool-workflows` from the marketplace

That's it. Both `/refine-tickets` and `/eng-team` are now available. Updates to the repo are picked up automatically.

### Alternative: Clone and add locally

If you prefer a local checkout:

```bash
git clone https://github.com/CoolAssTools/CoolAssTools.git
```

Then in Claude Code:
1. Run `/plugins`
2. Choose "Add marketplace"
3. Enter the local path to your clone

### Manual (copy command files)

If you only need `/refine-tickets` and don't want the full plugin:

```bash
cp plugins/cool-workflows/commands/refine-tickets.md ~/.claude/commands/
```

Note: `/eng-team` requires the `agents/` and `references/` directories — use the marketplace install for that one.

## Usage

### `/refine-tickets` — Ticket Refinement

Takes rough PM input and produces engineering-ready Asana tickets with user stories, personas, and acceptance criteria.

**How to use it:**

```
# Refine an existing Asana ticket
/refine-tickets https://app.asana.com/0/1234567890/9876543210

# Refine all tickets in an Asana project
/refine-tickets https://app.asana.com/0/1234567890

# Create tickets from a Google Doc
/refine-tickets https://docs.google.com/document/d/abc123/edit

# Create tickets from a description you type or paste
/refine-tickets We need a way for managers to export timecards to CSV
```

**What happens:**

1. It reads your input (fetches from Asana/Google Docs, or uses your pasted text)
2. Identifies who the feature is for (personas) and asks you to confirm
3. Analyzes for gaps — missing intent, vague requirements, unclear scope
4. Asks you clarifying questions one at a time
5. Composes structured tickets with user stories and acceptance criteria
6. Shows you everything for approval before writing
7. Updates/creates tickets in Asana

If Asana rate-limits the API, it saves everything to a local markdown file so nothing is lost.

**Who should use this:** Product managers, engineers, anyone turning a feature idea into actionable tickets.

---

### `/eng-team` — Engineering Team Orchestration

Spins up a full AI engineering team that plans, designs, and implements a feature — all tracked in Linear.

**How to use it:**

```
# From a text description
/eng-team https://linear.app/your-org/team/ENG/all Build a CSV export feature for timecards

# From a Linear ticket
/eng-team https://linear.app/your-org/team/ENG/all https://linear.app/your-org/issue/ENG-123

# From a Figma design
/eng-team https://linear.app/your-org/team/ENG/all https://www.figma.com/design/abc123/My-Design

# From an Asana ticket
/eng-team https://linear.app/your-org/team/ENG/all https://app.asana.com/0/project/task
```

The first argument is always your Linear team URL. The second is the feature context.

**What happens:**

1. **Product Discovery** — A ProductMgr agent asks clarifying questions and creates user story tickets in Linear
2. **Architecture** — Software Architect and Engineering Lead agents explore your codebase and design the approach
3. **Engineering Design** — Tickets get detailed implementation plans, dependencies, and backend/frontend splits
4. **Implementation** — Backend and Frontend engineer agents build the feature using TDD, each in isolated git worktrees

You approve each phase before it proceeds. The whole thing is tracked in Linear.

**Who should use this:** Engineers who want to go from feature description to working code.

---

## Typical Workflow

These two commands chain together naturally:

1. PM writes a rough feature doc in Google Docs or Asana
2. Run `/refine-tickets` to turn it into clean, well-structured Asana tickets
3. Pull the refined tickets into Linear
4. Run `/eng-team` to plan and build the feature

## Troubleshooting

**"Command not found"** — Make sure the plugin is installed. Run `/plugins` in Claude Code to check.

**Asana/Linear errors** — Make sure the respective MCP plugins are installed and authenticated. Run `/plugins` to verify.

**Rate limiting** — `/refine-tickets` automatically falls back to a local markdown file. For `/eng-team`, Linear rate limits are rare but you can retry.
