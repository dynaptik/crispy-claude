---
name: start
description: "Start the CRISPY workflow. Use when you want to build a new feature or vertical slice using the QRSPI methodology."
argument-hint: "Describe the feature or vertical slice you want to build"
---

# CRISPY Start

Initiates the CRISPY Orchestrator. This skill runs in the main conversation and orchestrates the Question → Research → Design auto-chain, stopping at the design human gate.

## Ask-Loop Pattern (MANDATORY for every subagent invocation)

Subagents cannot call `AskUserQuestion` — only the parent can. Wrap **every** subagent invocation in this loop:

1. Invoke the subagent.
2. Inspect the **first line** of its return summary:
   - If it begins with `STATUS: COMPLETE` → the phase artifact is done. Continue to the next step.
   - If it begins with `STATUS: NEEDS_USER_INPUT` → enter the ask sub-loop:
     a. Locate the `<questions>...</questions>` block in the return. Parse the JSON array inside.
     b. If `AskUserQuestion`'s schema is not yet loaded in this session, run `ToolSearch(query="select:AskUserQuestion")` once to load it.
     c. Call `AskUserQuestion` with the parsed array as the `questions` parameter (it maps 1:1).
     d. Re-invoke the **same subagent** with the original prompt PLUS a new `## User Answers` section appended:
        ```
        ## User Answers (from prior invocation)

        - **<question text>:** <user-selected label> (notes: <any>)
        - **<question text>:** <user-selected label>
        ```
        (If the user typed an "Other" answer, capture the custom text as the label and include any annotation notes.)
     e. Loop back to step 2.
   - If the first line is anything else, treat as a malformed return: report to the user and stop.

Trust the subagent's return summary. Do NOT read its working files to second-guess `STATUS`.

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

Invoke the `crispy-questioner` subagent under the Ask-Loop Pattern above. The subagent writes `.crispy/02_questions.md`. Any user-owned questions are surfaced as a `<questions>` block — you (the parent) ask the user via `AskUserQuestion` and re-invoke until `STATUS: COMPLETE`.

### 4. Phase R — Researcher

Invoke the `crispy-researcher` subagent under the Ask-Loop Pattern. It writes `.crispy/03_research.md` without reading `01_task.md` (context firewall). The Researcher should rarely need user input but the loop still applies.

### 5. Phase D — Architect

Invoke the `crispy-architect` subagent under the Ask-Loop Pattern. It writes `.crispy/04_design.md` and updates `adr/<iteration-name>.adr.md`. Expect at least one `STATUS: NEEDS_USER_INPUT` round for mandatory validation categories (tooling, packaging strategy, etc.).

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
