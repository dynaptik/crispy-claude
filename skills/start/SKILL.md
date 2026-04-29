---
name: start
description: "Start the CRISPY workflow. Use when you want to build a new feature or vertical slice using the QRSPI methodology."
argument-hint: "Describe the feature or vertical slice you want to build"
---

# CRISPY Start

Initiates the CRISPY Orchestrator. This skill runs in the main conversation and orchestrates the Question → Research → Design auto-chain, stopping at the design human gate.

## Steps

Follow these steps in order. Do not skip, reorder, or add phases beyond step 6.

### 1. Clean up previous run state

Use `Bash` to run this single command. Errors from missing worktrees/branches/files are suppressed:

```
git worktree remove .crispy-worktree --force 2>/dev/null; git branch -D crispy/implementation 2>/dev/null; mkdir -p .crispy && rm -f .crispy/*.md .crispy/*.log
```

### 2. Write the task file

Create `.crispy/01_task.md` with the user's intent from `$ARGUMENTS`:

```md
# Task

<verbatim contents of $ARGUMENTS>
```

Do NOT rewrite, summarize, or reformat the user's words. If `$ARGUMENTS` is empty, tell the user to re-run with a feature description and stop.

### 3. Phase Q — Questioner

Invoke the `crispy-questioner` subagent. Wait for it to return. The subagent writes `.crispy/02_questions.md`. If it collected user-owned inputs via `AskUserQuestion`, those interactions happened inside the subagent context — the main thread only sees the completion summary.

### 4. Phase R — Researcher

Invoke the `crispy-researcher` subagent. Wait for it to return. The subagent writes `.crispy/03_research.md` without reading `01_task.md` (context firewall).

### 5. Phase D — Architect

Invoke the `crispy-architect` subagent. Wait for it to return. The subagent writes `.crispy/04_design.md`, updates `adr/<iteration-name>.adr.md`, and may call `AskUserQuestion` during its work.

### 6. Human gate — STOP

Do NOT invoke the Structurer. Do NOT continue the workflow automatically.

Present a concise summary of `.crispy/04_design.md` (the Architect's return summary is fine). End with this exact closing:

> If changes are needed, reply with revision instructions for `04_design.md`.
> To continue the workflow, run `/crispy:resume`.

If the user replies with revision instructions after the gate, re-invoke the `crispy-architect` subagent to update `04_design.md` and the ADR, then present the same gate again.

## Important

- Do NOT read the plugin's own `agents/` or `skills/` directories. Subagents are invoked by name — Claude Code loads their instructions automatically.
- Do NOT list or verify the `.crispy/` directory after creating files in it. Trust the subagent return summaries.
- Each subagent runs in its own context window. The main thread only retains compact return summaries, not full subagent transcripts.

## Usage

```
/crispy:start Implement rate limiting for the /v1/auth endpoint
```
