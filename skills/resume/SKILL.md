---
name: resume
description: "Resume CRISPY from the last validated checkpoint. Use after approving the design, after a plan is written, between slices, or to finish a run."
---

# CRISPY Resume

Runs in the main conversation and dispatches to the next phase based on the highest-numbered artifact in `.crispy/`. This skill auto-chains S → P when the design is approved, but runs the Builder one slice at a time to preserve per-slice context flushing.

## Steps

### 1. Detect state

Use `Read` or `Glob` to find which of these exist under `.crispy/`:

- `01_task.md`, `02_questions.md`, `03_research.md`, `04_design.md`,
  `05_structure.md`, `06_plan.md`, `07_implementation.log`, `08_changelog.md`

The highest-numbered artifact is the current checkpoint.

### 2. Dispatch

Act based on the current checkpoint:

- **`08_changelog.md` exists** — The run is concluded. Tell the user: "This CRISPY run is concluded. Start a new run with `/crispy:start`." Stop.

- **`07_implementation.log` is latest** — A Builder run is in progress.
  1. Read only the headings and any `## Wrap Up` section.
  2. Count completed slices in the log and compare against the total in `.crispy/06_plan.md`.
  3. If all slices are complete and `## Wrap Up` is missing, invoke `crispy-builder` one more time to perform the wrap-up (full test suite, merge if worktree, `08_changelog.md`).
  4. Otherwise, invoke `crispy-builder` for the next slice. Stop after it returns.

- **`06_plan.md` is latest** — Implementation has not started yet. Invoke `crispy-builder` for slice 1. Stop after it returns.

- **`05_structure.md` is latest** — Auto-chain S+P is partially done (unexpected state). Invoke `crispy-planner`. After it returns, stop and tell the user to run `/crispy:resume` to start Implementation.

- **`04_design.md` is latest** — This is the post-design-approval path. Auto-chain the Structurer and Planner:
  1. Invoke `crispy-structurer`. Wait for it to return. It writes `.crispy/05_structure.md`.
  2. Invoke `crispy-planner`. Wait for it to return. It writes `.crispy/06_plan.md` and may call `AskUserQuestion` about git init and worktree creation.
  3. Stop. Tell the user: "Plan is ready. Run `/crispy:resume` to start Implementation (one slice per resume)."

- **`03_research.md` is latest** — The `/crispy:start` auto-chain did not complete Phase D. Invoke `crispy-architect` and follow the same human gate as `/crispy:start` step 6.

- **`02_questions.md` is latest** — The `/crispy:start` auto-chain did not complete Phase R. Invoke `crispy-researcher`, then `crispy-architect`, then present the design human gate.

- **`01_task.md` is latest** — The `/crispy:start` auto-chain did not complete Phase Q. Re-run `/crispy:start` semantics: invoke Questioner → Researcher → Architect, then human gate.

- **No artifacts** — Tell the user: "No CRISPY run in progress. Start one with `/crispy:start <task description>`." Stop.

## Important

- The Builder is invoked one slice at a time to preserve context flushing. Do NOT auto-chain multiple slices in a single `/crispy:resume`.
- If the user replies to the design gate with revision instructions instead of running `/crispy:resume`, re-invoke `crispy-architect` to update `04_design.md`, then present the gate again.
- Each subagent invocation runs in its own context window. Trust return summaries.
