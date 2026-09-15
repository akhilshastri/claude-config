# `/learn-and-code` templates

Read by the `/learn-and-code` command at scaffold time. The templates are copied into
the project's own `learnings/.templates/` so a project can diverge from these, and so
this file never has to be loaded again for the rest of the build.

---

## `CLAUDE.md` (project root)

~~~markdown
# <Project Name>

<One-paragraph description of what this project is.>

## How this project is being built

This project is built with the `/learn-and-code` workflow: an interactive,
step-by-step build where every step is explained and discussed before it is
written, reviewed after, and recorded.

- **The plan** lives in [plan/NN-<topic>.md](plan/NN-<topic>.md). Steps are ticked
  off as they are completed.
- **The learnings** live in [learnings/](learnings/), one file per step, in order.
  Start at [learnings/00-index.md](learnings/00-index.md).
- **The concept ledger** is [learnings/00-concepts.md](learnings/00-concepts.md) —
  every concept introduced so far. Grep it before explaining anything.

### The loop, for any agent working here

Recall → explain → predict → discuss ⛔ → implement → your turn ⛔ → walk through ⛔
→ record → commit? ⛔ → next.

The ⛔ marks are hard stops that wait for the user. Do not write code before the
discussion gate, and keep each step to one concept and roughly 150 lines of diff.
Never re-explain a concept already in the ledger — link to where it was introduced.
Run `/learn-and-code` with no arguments to resume at the next unticked step.

## Stack

<Language, framework, key libraries, package manager, test runner.>

## Commands

| Task | Command |
|---|---|
| Install | `<...>` |
| Dev | `<...>` |
| Test | `<...>` |
| Build | `<...>` |
~~~

---

## `learnings/00-index.md`

~~~markdown
# Learning Index — <Project Name>

Built with `/learn-and-code`. Each step below was explained, discussed, implemented,
reviewed and recorded in order. Read top to bottom for the full story, or jump to a
step by its takeaway.

Plan: [plan/NN-<topic>.md](../plan/NN-<topic>.md) · Concepts: [00-concepts.md](00-concepts.md)

| # | Step | Takeaway | Concepts | Predicted | Confidence | Notes |
|---|------|----------|----------|-----------|------------|-------|
| 01 | <title> | <the thing worth remembering> | <n> | ✓ / ✗ / — | got it | [notes](01-<slug>.md) |

`Predicted`: ✓ the user's approach matched, ✗ it differed, — they skipped.
`Confidence`: the user's own word at the end of the step — got it / fuzzy / lost.
~~~

---

## `learnings/00-concepts.md`

~~~markdown
# Concepts — <Project Name>

Every concept introduced, in order of first appearance. Two jobs: it is the first
thing to grep when returning to this project, and it is the list the build reads to
know what it must *not* re-explain.

| Concept | First seen | In one line | Revisit? |
|---|---|---|---|
| <name> | [step 03](03-<slug>.md) | <definition, one line> | — |

`Revisit?` is marked when the user ends a step on *fuzzy* or *lost*. Marked concepts
are the pool the next steps draw their recall question from; clear the mark once the
user answers it cleanly.
~~~

---

## `learnings/.wip/NN-notes.md` (working file)

Created at the start of each step, appended to throughout, deleted at stage 6 once the
learning file is written. This is the anti-compaction device: anything that only exists
in conversation is on a timer, so it goes here the moment it happens.

~~~markdown
# Step NN notes — working file

## Prediction
> <the user's own words, verbatim — or `skipped`>

## Questions
- **Q:** <verbatim, the user's own wording>
  **A:** <what the answer established, one or two lines>

## Decisions
- <what was being decided> → **<chosen>** (rejected: <alt>) — <why>

## Gotchas hit
- <symptom first, then cause>

## Your turn
- Handed over: <what and why it was a real decision>
- They wrote: <approach in one line> · Kept as-is / adjusted: <what and why>

## Confidence
<got it | fuzzy | lost> — <which concept, if not "got it">
~~~

---

## `learnings/NN-<step>.md`

~~~markdown
# Step NN — <Title>

**Date:** YYYY-MM-DD · **Plan:** [step NN](../plan/NN-<topic>.md) · **Status:** complete
**Builds on:** [step 03](03-<slug>.md) · [step 05](05-<slug>.md)
**Referenced later by:** <backfilled when a later step leans on this one>

## Goal

<One paragraph: what this step achieves, and why it belongs at this point in the build.>

## New concepts

- **<Concept>** — <what it is, in one line> · <why it applies here>

## Decisions

| Decision | Chosen | Alternatives rejected | Why |
|---|---|---|---|
| <what was being decided> | <what we did> | <what we didn't> | <the reasoning> |

## Prediction vs implementation

<Only if the user predicted. Where they were right, where it would have bitten them.
Be honest in both directions — this is usually the most valuable part of the file.>

## Questions I asked

Questions verbatim; answers distilled. The user's own wording is what they will
recognise in six months.

- **Q:** <exactly as asked>
  <what the answer established>

## Your code

<Only if stage 4a ran. What was handed over, what the user wrote, and whether it was
kept as-is or adjusted — with the reason.>

## Code map

| File | What it does |
|---|---|
| [path/to/file.ts:42](path/to/file.ts#L42) | <role in this step> |

## Gotchas

- <Symptom first, then cause. The symptom is what you will recognise next time.>

## Recap

1. <Question about the core mechanism>
2. <Question about why an alternative was rejected>
3. <Question about a gotcha>

<details>
<summary>Answers</summary>

1. <answer>
2. <answer>
3. <answer>

</details>
~~~

### Appended later, when a decision here is overturned

~~~markdown
## Superseded

- <which decision> was replaced in [step 09](09-<slug>.md) — <what changed and why>.
~~~
