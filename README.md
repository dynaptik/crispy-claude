<!-- LOGO -->
<h1 align="center">
    <img src="https://github.com/user-attachments/assets/a00e2bee-3f0a-409e-93d5-0b63458317df" alt="CRISPY logo" width="512">
    <br />
    CRISPY Orchestrator for Claude Code
</h1>

<p align="center">
    Opinionated multi-agent orchestration plugin for software development.
    <br />
    QRSPI-inspired staged workflow as a native Claude Code plugin.
    <br />
    <a href="#core-principles">About</a>
    ·
    <a href="#installation--setup">Install</a>
    ·
    <a href="#usage">Usage</a>
    ·
    <a href="#orchestration-flow">Flow</a>
    ·
    <a href="#the-8-phases">Workflow</a>
    ·
    <a href="#license">License</a>
</p>

CRISPY (how QRSPI sounds to me phonetically) is a multi-agent orchestration plugin for software development. It implements an opinionated staged workflow around the QRSPI methodology (Question, Research, Structure, Plan, Implement).

This is the Claude Code port. The original VS Code GitHub Copilot version lives at <https://github.com/dynaptik/crispy>.

Its purpose is to make agentic coding more reliable while also being heavily opinionated about which architecture style lends itself to agentic coding (vertical slices or moduliths, alternatively) and how it should be packaged (plugin architecture rather than scavenging for `.md` files all the time).

Note: while some things like the 40 instructions limit seem arbitrary, they are grounded in instruction-to-weight ratios which are often described in the community as limits when models start to "drift". Feel free to question some choices by opening issues to discuss it.

## Core Principles

* **Context Firewalls:** Subagents are isolated by instruction boundaries. The Researcher is forbidden from reading `01_task.md` via prompt-level enforcement — it only sees `02_questions.md`. This is a logical firewall (the model respects the instruction), not a tool-level restriction.
* **Vertical Slices:** Development is broken into independent, testable slices that deliver one narrow capability end-to-end (logic + interface + tests) rather than horizontal architectural layers. This is a planning and execution rule, not a guarantee that every repo will use a `src/features/` folder scheme.
* **Instruction Budgeting:** Each agent is authored with fewer than 40 instructions (self-documented via `Budget: N/40` annotations). This is an authoring discipline to stay below empirical drift thresholds, not a runtime enforcement mechanism.
* **Enforced Context Flushing:** The Builder agent is instructed to exit after each slice. The user triggers the next slice via `/crispy:resume`, which reinvokes the Builder with a clean context window. Earlier phases run in their own subagent contexts, so the main thread only sees their compact return summaries. See [Orchestration flow](#orchestration-flow) for which phases auto-chain and which stop for user input.

## The 8 Phases

1. **Question:** Decompose intent, collect any missing user-owned constraints, and write `02_questions.md`
2. **Research:** Objective codebase analysis; outputs `03_research.md`
3. **Design:** Architectural alignment; outputs `04_design.md` (Human Gate)
4. **Structure:** Map vertical implementation slices; outputs `05_structure.md`
5. **Plan:** Tactical task list and checkboxes; outputs `06_plan.md`
6. **Repository Setup:** Optionally initialize git, then optionally create a worktree for isolated execution
7. **Implement:** Execute slices one-by-one, flushing context after each slice
8. **Wrap Up:** Write `08_changelog.md` with a concise implementation summary and merge status

## Installation & Setup

### Requirements
* **Claude Code** installed and authenticated
* **Git** — optional, but recommended for commits, worktrees, and clean iteration history. CRISPY can still run without git if you choose not to initialize a repository.

### Setup

No build step required. The plugin is pure markdown and JSON.

#### Option A — Local testing with `--plugin-dir`

```bash
git clone https://github.com/dynaptik/crispy-claude
claude --plugin-dir ./crispy-claude
```

#### Option B — Install from a marketplace

Publish or install through a Claude Code plugin marketplace, then enable with:

```
/plugin install crispy@<marketplace-name>
```

#### Option C — Quick local install

Copy the plugin directory into a local marketplace, then run `/plugin install crispy`. See the [Claude Code plugin marketplace docs](https://code.claude.com/docs/en/plugin-marketplaces) for details.

## Usage

Start the pipeline with the `/crispy:start` skill:

```
/crispy:start Implement rate limiting for the /v1/auth endpoint
```

### Skills
* `/crispy:start`: Begin the QRSPI pipeline with a feature description
* `/crispy:status`: View artifact state, likely current phase, worktree mode, and the next expected action
* `/crispy:resume`: Pick up from the last validated checkpoint, or continue after an approved design

### Agents

The plugin provides 6 specialized subagents that form the handoff chain:

**Questioner** → **Researcher** → **Architect** → *(human gate: Architect asks for approval, then the user explicitly continues)* → **Structurer** → **Planner** → **Builder**

Each agent has restricted tool access and a scoped instruction budget. The Researcher is explicitly forbidden from reading `01_task.md` (context firewall). The Builder exits after one slice (context flushing).

Phase Q may ask the user a small number of structured clarification questions (via `AskUserQuestion`) when core requirements are still missing. Those answers are treated as confirmed inputs. `02_questions.md` is for the Researcher, not a questionnaire the user is expected to answer manually.

When the Architect finishes `04_design.md`, it stops and presents a single approval gate: reply with revision instructions if changes are needed, or continue the workflow with `/crispy:resume`.

### Handoff model

Claude Code has no "handoff button" UI like VS Code Copilot. Instead, the `/crispy:start` and `/crispy:resume` skills act as **main-thread orchestrators**: their bodies tell the main conversation which subagent to invoke next, wait for the subagent to return, and then either chain to the next phase or stop.

Each subagent runs in its own isolated context window. The main thread only accumulates compact return summaries, not full subagent transcripts.

### Orchestration flow

This Claude Code port groups the phases into larger auto-chained blocks than the original VS Code plugin. The user interaction points are the only places the flow stops.

| Skill invocation | Auto-chained phases | Stops at |
|---|---|---|
| `/crispy:start <task>` | Q → R → D | Design human gate |
| `/crispy:resume` (after design approval) | S → P | Before Implementation |
| `/crispy:resume` (after plan, or between slices) | one Builder slice | After each slice |
| `/crispy:resume` (after last slice) | Builder wrap-up | Run concluded |

Rationale for the grouping:

* **Q → R → D is safe to auto-chain.** The Questioner collects any user-owned inputs via `AskUserQuestion` inside its own context, the Researcher runs blind of `01_task.md`, and the Architect may also call `AskUserQuestion` for infrastructure validation. The main thread only sees three compact return summaries before the design gate.
* **S → P is safe to auto-chain.** Both are small, deterministic transformations over `04_design.md` and `05_structure.md`. The Planner still asks its git-init and worktree questions via `AskUserQuestion` inside its own context.
* **Builder is never auto-chained across slices.** Each slice is run in a separate `/crispy:resume` invocation so the main thread flushes the previous slice's summary before planning the next one. This preserves the "enforced context flushing" principle from the original CRISPY design.

### Deviations from the VS Code plugin

| Area | VS Code plugin | This Claude Code port |
|---|---|---|
| Phase handoffs | `handoffs:` YAML with `send: false` per agent | Main-thread orchestration from skill bodies |
| User action per phase | One handoff button click per phase | One `/crispy:resume` per **block** (Q+R+D, S+P, or one slice) |
| Start behavior | `/crispy:start` runs only Phase Q, stops for handoff | `/crispy:start` auto-chains Q → R → D, stops at design gate |
| Post-design continuation | User clicks three handoff buttons (S, P, then Builder) | `/crispy:resume` auto-chains S → P, then stops before Builder |
| Builder slice cadence | One handoff click per slice | One `/crispy:resume` per slice (unchanged) |
| Agent frontmatter | `user-invocable: false`, `handoffs:` list | kebab-case `name`, Claude tool names, no handoff fields |
| User question tool | `vscode/askQuestions` | `AskUserQuestion` (native) |

The net effect is fewer user interactions for the same workflow, while preserving the two interactions that matter: the design human gate and the per-slice context flush.

## CRISPY Project Tree

### `.crispy/` working state
This folder is ephemeral scratch space for the current pipeline run. Each `/crispy:start` overwrites the previous artifacts. By naming files 01_ through 08_, any developer (or a resumed AI agent) can see the logical progression of the *current* feature. If the AI hallucinates during implementation, you can point it back to `04_design.md` as the ground truth.

Once the iteration is concluded, these files have served their purpose — the decision history is preserved in your git log if you are using git, plus the final `08_changelog.md` summary. If you want to keep them, commit before starting a new run.

### `.crispy-worktree/` (optional)
If the project is not yet a git repo, the Planner asks whether it should initialize one first. Only after the repo is confirmed or initialized does it offer `.crispy-worktree/` as an isolated checkout. On a new `/crispy:start` run, any existing worktree is cleaned up automatically. The Builder works inside the worktree to keep the main branch clean during implementation.

### Physical package layout
CRISPY does not mandate a single folder layout. The important invariant is that slices are planned and implemented end-to-end; the physical packaging style is an explicit design choice.

* **Existing codebase:** Preserve the dominant package boundaries discovered in Research and confirmed in Design unless there is a deliberate, justified reason to change them.
* **Greenfield or new subsystem:** Design must explicitly choose a packaging style up front (for example: co-located feature folders, a stage-centric modulith, or plugin/module packages) and justify why it fits the repo.
* **`src/features/` is one valid expression, not the definition of CRISPY.**

Illustrative greenfield TypeScript example:

```
my-typescript-service/
├── .crispy/                        # CRISPY ARTIFACTS (State & Context)
│   ├── 01_task.md                  # raw user intent (this is your nucleus!)
│   ├── 02_questions.md             # socratic inquiries (The "Q" Phase)
│   ├── 03_research.md              # blind codebase facts (The "R" Phase)
│   ├── 04_design.md                # approved architecture (The "Brain Surgery")
│   ├── 05_structure.md             # vertical slice definitions (Checkpoints)
│   ├── 06_plan.md                  # tactical task-by-task execution list
│   ├── 07_implementation.log       # verification results for each slice
│   └── 08_changelog.md             # concise run summary and merge status
│
├── src/                            # PROJECT SOURCE (Vertical Slice Style)
│   └── features/                   # core business logic grouping
│       └── auth-rate-limiting/     # a completed CRISPY vertical slice (ts-example)
│           ├── middleware.ts       # route protection logic
│           ├── redis-store.ts      # storage implementation
│           ├── types.d.ts          # feature-specific types
│           └── __tests__/          # integrated slice tests
│               └── rate-limit.test.ts
│
└── .gitignore
```

## Plugin layout (for plugin authors)

```
crispy-claude/
├── .claude-plugin/
│   └── plugin.json           # plugin manifest
├── agents/                   # 6 subagent definitions
│   ├── architect.md
│   ├── builder.md
│   ├── planner.md
│   ├── questioner.md
│   ├── researcher.md
│   └── structurer.md
├── skills/                   # 3 slash commands
│   ├── resume/SKILL.md
│   ├── start/SKILL.md
│   └── status/SKILL.md
├── LICENSE
└── README.md
```

## The opinionated parts
* Implementation is a `.log` rather than a `.md` file since it contains stdout and can easily grow too large. To preserve "instruction budget" I try to avoid digestion by the agent. Logfiles are usually not picked up, except there is an explicit need to debug something.
* When talking about execution sandbox and isolation I mostly talk about keeping the context and "focus" of agents sharp and avoiding pollution. Additionally I urge you to use actual sandboxing/isolations like devcontainers, microVMs, bubblewrap/seatbelt or whatever your heart desires. I'm personally a fan of [https://github.com/always-further/nono](https://github.com/always-further/nono) — YTMMV, your threat model may vary.

## Credits
This project is grounded in the research by Dexter Horthy, the articles of Alex Lavaee, and the QRSPI prompt engineering work by Matan Shavit. I (Danijel Milicevic — dynaptik) just did some legwork and put it all together, simplified some things, and tested it across multiple personal and professional projects (which is ongoing).

## License
MIT
