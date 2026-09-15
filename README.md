# claude-config

My personal [Claude Code](https://claude.com/claude-code) configuration: global
instructions, agent personas, custom slash commands and skills.

This repo lives at `~/.claude`, which also holds credentials, conversation
transcripts and a large plugin cache. [.gitignore](.gitignore) is therefore
**deny-by-default** — everything is ignored and only the files below are
explicitly allowed back in. To version something new, add an explicit `!path`
rule; never remove the leading `*`.

## Contents

| Path | What it is |
|---|---|
| [CLAUDE.md](CLAUDE.md) | Global instructions applied to every project — commit conventions, planning workflow, tooling and dependency preferences, code style, and the Windows/WSL2 + Chrome environment notes. |
| [settings.json](settings.json) | Model, theme, and enabled plugins. |
| [agents/](agents/) | Agent personas for planning, architecture, UX, implementation, QA, review, and learning capture. |
| [commands/](commands/) | Custom slash commands. |
| [skills/](skills/) | Personal skills: AG Grid, AMPS, FlexLayout, WSL Chrome debugging. |

## Multi-agent workflow

This repo is organized around a small team model instead of a single general-purpose agent. The workflow is intentionally staged so each role owns one part of the system and the handoff is explicit.

### 1. Plan before implementation

The `planner` agent is the first stop for any non-trivial task. It produces a scoped, ordered plan with milestones, sequencing, and dependencies. The rule in [CLAUDE.md](CLAUDE.md) is simple: if something is substantial, start with planning and save the plan under `plan/xx-yyy-yyy.md`.

### 2. Stress-test the plan

The `devil` agent acts as the adversarial reviewer. It looks for hidden assumptions, edge cases, security concerns, and reasons a proposal may fail. This is the safety check before implementation begins.

### 3. Design the structure and experience

Two design roles split the work:

- `architect` decides the system structure, module boundaries, data flow, and technology choices.
- `dx-ux-designer` defines the user and developer experience: flows, naming, API/CLI shape, layouts, states, and interaction design.

The intent is to separate “what the system should look like” from “how it is built.”

### 4. Implement the scoped change

The `coder` agent turns the design and plan into working code. It is expected to stay within scope, respect the project's existing patterns, avoid speculative refactors, and rely on the selected framework/library conventions rather than inventing a parallel approach.

### 5. Verify in reality

The `qa` agent focuses on proof, not assumption. It writes tests and verifies the change by running the project and, when relevant, exercising the app in a real browser with the Chrome DevTools tools. It is responsible for reporting what was actually tested and what remains unverified.

### 6. Record the learning

The `scribe` agent is used by the `/learn-and-code` flow to turn raw working notes into a structured learning file. This keeps the project’s guided teaching process durable: the code changes and the reasoning behind them are captured together.

### End-to-end loop

The repo’s main orchestrator is the `/learn-and-code` command in [commands/learn-and-code.md](commands/learn-and-code.md). Its loop is:

- explain the concept
- ask for a prediction
- discuss trade-offs
- implement
- review the diff
- record learning
- optionally commit the code and notes together

This makes the workflow educational as well as productive: each step is explained, checked, and retained for later review.

## Agent personas in this repo

| Agent | Role |
|---|---|
| `planner` | Breaks work into ordered, reviewable tasks. |
| `architect` | Chooses structure, boundaries, and technology. |
| `dx-ux-designer` | Defines UX/DX flows and interface contracts. |
| `coder` | Implements the agreed solution. |
| `qa` | Validates behavior with tests and browser-driven verification. |
| `devil` | Finds failure modes and pushes back on weak assumptions. |
| `scribe` | Converts guided notes into durable learning docs. |

## Commands

### `/learn-and-code`

Builds an application as a guided learning session rather than a code dump.
Every step is explained and discussed *before* it is written, reviewed after,
and then recorded to `learnings/NN-<step>.md` so the build can be recapped
months later.

The loop, with hard stops that wait for the user:

```
explain → predict → discuss ⛔ → implement → walk through ⛔ → record → commit? ⛔ → next
```

Run with a scenario to start (`/learn-and-code build a trading dashboard in React`),
or with no arguments to resume at the next unticked step of the current plan.
See [commands/learn-and-code.md](commands/learn-and-code.md).

## Setup on a new machine

```bash
git clone <this-repo> ~/.claude-config
# copy or symlink the tracked paths into ~/.claude
```

Plugins, credentials and history are intentionally not included — sign in with
`claude` and re-enable plugins from `settings.json`.
