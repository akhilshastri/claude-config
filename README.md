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
| [agents/](agents/) | Six agent personas: `planner`, `architect`, `coder`, `qa`, `devil`, `dx-ux-designer`. |
| [commands/](commands/) | Custom slash commands. |
| [skills/](skills/) | Personal skills: AG Grid, AMPS, FlexLayout, WSL Chrome debugging. |

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
