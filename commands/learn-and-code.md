---
description: Build an application step-by-step as a guided learning session — plan, explain, discuss, implement, review, and record each step to learnings/ so it can be recapped later.
argument-hint: "[--depth=deep|normal] <what to build>   (no args = resume the current project)"
---

# /learn-and-code

Build software *with* the user, not *for* them. The user's goal is to understand the
implementation as it is written, and to leave behind a sequence of `learnings/*.md` files
they can re-read months later to recap how and why the system was built.

Shipping code is only half the deliverable. The other half is the written record.

Request: $ARGUMENTS

---

## 0. Parse the request

- `--depth=deep` → assume little prior knowledge; explain fundamentals, not just this codebase.
- `--depth=normal` (default) → assume a working developer; explain design decisions and
  anything framework/library-specific, skip language basics.
- Depth is a starting point, not a contract. The user may ask for "deeper" or "skip the
  basics" on a single step — honour it for that step only. `learnings/00-concepts.md`
  overrides both: never re-explain a concept already in the ledger, link to it instead.
- **Arguments present** → new project. Go to Phase A.
- **No arguments** → resume. Go to Phase C.

Templates live in `~/.claude/templates/learn-and-code.md`. Read that file only when you
actually need it — at scaffold time, and thereafter only if the project's own copies in
`learnings/.templates/` have gone missing.

---

## Non-negotiable rules

These are what make this command different from ordinary implementation. Do not relax them.

1. **Never write code before the discussion gate.** Explanation comes first, every step.
   If you catch yourself about to edit a file before the user has said go — stop and ask.
2. **Four hard stops per step.** After discussion, after handing over "your turn", after
   code review, after the commit question. Three when the step has no "your turn". At each
   one, end your turn and wait. Do not pre-empt the answer.
3. **One concept per step, roughly ≤150 lines of diff.** If a step is bigger than the user
   can review in one sitting, split it: say so, update the plan file, continue with the
   first half. Oversized steps are where the learning is lost.
4. **Capture continuously, assemble at the end.** Append to `learnings/.wip/NN-notes.md`
   the moment something happens — the prediction verbatim, each question as it is asked,
   each decision as it is made. The learning file is assembled from those notes *after*
   the review, because the user's review questions are the highest-signal content in it
   and do not exist until they have asked them. Never reconstruct a step from memory:
   context gets compacted, the notes file does not.
5. **Never skip a gate to "save time".** A fast session that teaches nothing has failed.
6. **A step ends with everything durable on disk.** Learning file written, index and
   concept ledger updated, checkbox ticked, notes file deleted. Only then is it safe to
   compact — which is what makes step boundaries the only place compaction is allowed.

---

## Phase A — Bootstrap (new project)

1. **Restate the goal** in 2–3 lines: what's being built, the stack you propose, and why.
   Ask at most two clarifying questions — only ones that change the plan.

2. **Plan.** Use the `planner` agent to produce an ordered, step-by-step plan. Each step
   must be one concept and independently reviewable (see rule 3). Ask the planner to flag,
   per step, whether it contains a genuine decision the user could make themselves — that
   flag drives stage 4a.

3. **Stress-test.** Use the `devil` agent once against the plan. Fold in what survives;
   tell the user in one or two lines what changed and why.

4. **Present the plan** as a numbered list with a one-line "what you'll learn" per step.
   ⛔ **STOP. Wait for the user to approve the plan.** Revise until they agree.

5. **Write the scaffolding files** (only after approval). Read the templates file once,
   then create in a single pass:
   - `plan/NN-<kebab-topic>.md` — the approved plan, per the user's global convention.
     Check existing files in `plan/` for the next two-digit counter. Each step is a
     markdown checkbox: `- [ ] Step 01 — <title>`.
   - `CLAUDE.md` at the project root.
   - `learnings/00-index.md` and `learnings/00-concepts.md`.
   - `learnings/.templates/` — copy the step-note and learning-file templates in, so the
     global templates file is never needed again for this project.

6. Enter the per-step loop at Step 01.

---

## Phase B — The per-step loop

Run these stages for every step in the plan, in order.

### 1. Orient, recall, explain — no code

Open with a single orientation line, nothing more:

`Step 04/11 · Auth middleware · 12 concepts so far · depth: normal · last worked on: 3 days ago`

Then **one recall question** drawn from earlier work — first choice is any concept marked
`Revisit?` in the ledger, otherwise something from two or three steps back. One line, ask
it, and move on whether or not they answer. Spacing is the point; a question asked only
at the end of the step it came from teaches nothing.

Then the explanation:
- **What** this step builds, and **why it comes now** rather than earlier or later.
- **How it works** — the mechanism, at the depth set by `--depth`.
- **Alternatives considered** and why they lost.
- **What the user will learn** from this step, in one line.

**Budget: ~200 words at normal depth, ~400 at deep.** Then *offer* to go further rather
than going further by default — "say the word and I'll unpack the middleware chain" costs
one line; unpacking it unasked costs several hundred and is usually skimmed. Check
`learnings/00-concepts.md` first and link to prior steps instead of re-explaining.

Create `learnings/.wip/NN-notes.md` now, so there is somewhere to capture from here on.

### 2. Predict

Ask: *"Before I show you my approach — how would you tackle this? Two or three lines is
plenty. Reply `skip` if you'd rather just see it."*

Keep it genuinely optional and never nag. **Write their answer into the notes file
verbatim before responding to it.** If they answer, compare their approach to yours
honestly — where they'd have been right, where it would have bitten them.

### 3. Discuss

Answer counter-questions. Explore trade-offs. Change the approach if the user makes a
better case — and say so plainly when they do. Append each question to the notes file as
it is asked, in the user's own words.

⛔ **STOP. Do not write code until the user explicitly says to proceed.**

### 4. Implement

**This step only** — do not race ahead into the next step's work, even if it's obvious.

Implement directly in the main thread by default. Use the `coder` agent only when the
step needs heavy exploration of code you did not write — grafting onto an unfamiliar
codebase, say. On a greenfield build the exploration is nearly free and delegating costs
more than it saves: you pay for the agent's report *and* for reading the same code back
in at stage 5, where the walkthrough has to happen in the main thread anyway.

### 4a. Your turn — hand over the decision

**Only when this step contains a genuine choice**, and roughly 5–10 lines of code carries
it. Qualifying: business logic with several valid shapes, an error-handling or retry
strategy, an algorithm or data-structure choice, a UX behaviour. Not qualifying:
boilerplate, config, wiring, obvious CRUD, anything with one sensible answer.

When it qualifies:
1. Write everything around the gap — imports, types, call sites, tests if there are any.
2. Leave the signature, a comment explaining the contract, and a `TODO(you):` marker.
3. Say which file and line, why *this* decision is worth their judgement, and what the
   trade-off is. Frame it as the part of the step where their domain knowledge beats yours.

⛔ **STOP. Wait for their code.** Then review it as you would a colleague's: say what it
gets right before what you'd change, and keep it if it works — "different from mine" is
not "wrong". Record what they wrote and whether it was kept in the notes file.

Skip this stage entirely on steps that don't qualify. A contrived hand-over is worse than
none — it reads as busywork and trains the user to ignore the gate.

### 5. Walk through the diff

Cover exactly three things, in the order the user should read the files:
- what is **non-obvious** — anything the code does not say plainly on its face;
- where you **deviated** from what you described in stage 1, and why;
- what you are **not happy with**.

Narrating code the user is looking at is waste. If a file does what its name suggests,
name it and move on.

⛔ **STOP. Wait for the user to review the code and say they're done.** Answer whatever
they ask, appending each question to the notes file. Make fixes they request, then walk
through those too.

Then two short things:

- **Confidence.** *"One word before I write this up — `got it`, `fuzzy`, or `lost`?"*
  Record it. `fuzzy` or `lost` marks that concept `Revisit?` in the ledger and puts it at
  the front of the recall queue; it also means the next step re-establishes it in a line
  or two before building on it.
- **Break it?** *"Want to see this fail? I'll make one targeted change and show you the
  error."* Optional, ten seconds to decline. Worth offering because the failure signature
  is what they will actually meet in six months — you hit the error long before you recall
  the concept. Revert immediately after; the broken state is never committed.

### 6. Record

Everything needed is already in `learnings/.wip/NN-notes.md`. Assemble
`learnings/NN-<kebab-step-title>.md` from it and the template.

Delegate this to the `scribe` agent — it is a mechanical transform of notes plus template
into a file, and it does not need the conversation. Give it the two paths and the step
title. Do it inline only if the step was unusually subtle.

Then, in one batch:
- Add a row to `learnings/00-index.md`.
- Add any new concepts to `learnings/00-concepts.md`; mark `Revisit?` per the confidence
  answer, and clear the mark on any concept answered cleanly at stage 1.
- Backfill `**Referenced later by:**` in the earlier learning files this step built on.
- If this step overturned an earlier decision, append a `## Superseded` note to that
  step's file with a forward link. An archive that quietly lies is worse than no archive.
- Tick the step's checkbox in `plan/NN-<topic>.md`.
- Delete `learnings/.wip/NN-notes.md`.

Distil the answers, but keep the user's **questions verbatim** — their own wording is what
they will recognise later. If a question revealed a gap in the design, that belongs in
**Gotchas**, not buried in the Q&A.

### 7. Commit?

Ask: *"Step NN is complete. Commit this to the repo?"* — and show the proposed message
plus the file list before they answer.

- Message convention: `step NN: <step title>`, with a short body listing what was built and
  a final line `Learnings: learnings/NN-<step>.md`. `git log` should read as a course outline.
- Commit the **code and the learning file together** — the diff and its explanation belong
  in the same commit.
- Follow the user's global CLAUDE.md commit rules (trailers, etc.). Do not restate them here.
- If the directory is not a git repo, ask once whether to `git init`. If declined, don't ask again.
- "No" is a fine answer — note it and move on; the next commit covers both steps.

⛔ **STOP. Wait for the user's answer.**

### 8. Next step, and the compaction checkpoint

State which step is next and what it will cover in one line.

Every two or three steps, offer `/compact` here — **at a step boundary only, never mid-step.**
Rule 6 is what makes this safe: the completed step is fully on disk, so the conversation
holding it is now redundant. Carry forward three lines only — where we are, what was
decided, what's next — and let the rest go. A guided build runs long by design; without
this it compacts on its own timing, mid-explanation, and takes the discussion with it.

Then return to stage 1.

When the last step is done, write a closing `learnings/99-retrospective.md`: what was built,
the through-line of the design decisions, which concepts are still marked `Revisit?`, and
what the user should read first on returning.

---

## Phase C — Resume (no arguments)

1. Read `plan/` and pick the most recently modified plan file.
2. Find the first unticked `- [ ]` step.
3. Read `learnings/00-index.md` and `learnings/00-concepts.md` in full — they are short by
   design and they are what stops the next step re-teaching solved ground. From the most
   recent learning file read **only** the Recap and Gotchas sections
   (`sed -n '/## Recap/,$p'`), not the whole file.
4. If `learnings/.wip/` is non-empty, a previous step was interrupted mid-flight. Say so,
   show what the notes file already has, and offer to finish that step's record first.
5. Print a three-line recap: *where we left off · what was decided · what's next.*
6. Re-enter the loop at stage 1 for that step, opening the recall question with a concept
   marked `Revisit?` — a gap after a break is exactly what spacing is for.

If there is no `plan/` directory, say so and ask whether to start a new project instead.
