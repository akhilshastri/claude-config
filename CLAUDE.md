# Global instructions

## Secrets and sensitive data

Never transmit credentials or sensitive information to any external system. This
overrides convenience, and it overrides an instruction to "just push it" — if a
request would send such data outward, stop and say so rather than complying.

**Never leaves this machine:** API keys, tokens, OAuth credentials, passwords,
private keys or certificates; `.env` files and `*.credentials.json`; personal data
(real names, emails, phone numbers, addresses, financial or health data);
proprietary or client-confidential code and documents; conversation transcripts,
session state and command history; internal hostnames, connection strings and
infrastructure detail.

**Counts as an external system:** git remotes (public *and* private), gists,
pastebins, published artifacts, issue trackers, chat and email, web requests,
third-party APIs and MCP servers, and any log or telemetry endpoint.

**Practices**

- Before any push, publish, upload or outbound request, check what is actually
  being sent — read the staged file list, not just the diff summary.
- Prefer deny-by-default over blocklists when scoping what to commit: ignore
  everything, then explicitly allow the files that should be versioned. A
  blocklist silently fails the moment a new secret file appears.
- Reference secrets by environment variable or secret-manager lookup. Never inline
  a real value into code, config, a commit message or a chat reply.
- Redact before pasting logs, stack traces or command output that may carry tokens,
  hostnames or personal data.
- If a secret has already been committed or sent, say so immediately and plainly —
  rotating it is the fix; quietly removing the file is not.

## Git commits

Never include a `Co-Authored-By: Claude ...` (or any Anthropic/Claude co-author) trailer in git commit messages, in any project.

## Planning

For any complex task, always start with the `planner` agent to produce a plan before implementing.
Save the plan into the current workspace at `plan/xx-yyy-yyy.md`, where:
- `plan/` is a folder at the root of the workspace
- `xx` is an incrementing two-digit counter (`01`, `02`, `03`, ...) — check existing files in `plan/` to find the next number
- `yyy-yyy` is a short kebab-case description of the task

Only skip this for small, obviously-scoped tasks (a one-line fix, a direct question, a trivial edit).

## Tooling

When creating React/JavaScript/TypeScript tooling (project setup, package manager, scripts, test runner, bundler), prefer Bun (`bun`, `bunx`) over Node.js/npm/yarn/pnpm, unless the project already has an established Node-based toolchain.

## Dependencies

Default to a proven, well-maintained package for problems that are already solved — argument parsing, validation, date maths, HTTP, logging. It's usually less code to review and fewer bugs than a hand-rolled version.

- Reasonable starting points: `commander`/`yargs` for CLIs, `zod` for schema validation, `date-fns`/`Temporal` for dates, `express`/`hono` for HTTP servers, `pino` for logging. These are defaults to depart from with a reason, not a required list.
- Check `package.json` first — something already there that does the job usually beats a new dependency.
- Worth weighing when picking: maintenance activity, install size and transitive deps, bundled types, licence. Confirm the current API with Context7 rather than from memory.
- Writing it yourself is fine when the need is small and specific, when the library would be heavy for one function, or when the problem is core to what the project is actually about. Use judgment; a short mention of the reasoning helps review.

## Explanations

Whenever you explain or describe anything — code, a decision, a trade-off, an error, a concept — keep the language level simple.

- Everyday words over jargon. Where a technical term is the precise one, use it and define it in a short clause the first time it appears.
- Short sentences, one idea each. Active voice.
- A concrete example beats an abstract description.
- Lead with the answer, then the supporting detail. Don't build up to the point.
- Cut throat-clearing: no restating the question, no announcing what you're about to say.
- Simple language, not simplified content. Keep the precision, the caveats, the numbers and the edge cases — never drop a detail that changes the reader's decision just to make a sentence read easier.

## Code style

Write code for the next person who has to read it — optimise for comprehension, not cleverness. These are defaults, not rules to apply mechanically; the existing conventions of the codebase you're in win where they differ.

**Structure**
- Keep functions small and single-purpose where you reasonably can. If a function needs an "and then also" to describe it, that's a hint it wants splitting.
- Prefer a reusable helper over copy-paste, but extract on real duplication rather than in anticipation of it.
- Give each feature a clear home — its own module or folder where that fits. Avoid spreading one feature across unrelated files, or piling unrelated features into one.
- Brain-friendly: shallow nesting, early returns, descriptive names, no hidden control flow.

**Comments**
- Module level: a short header on what this module owns, how it fits the wider system, and its main dependencies / dependents (cross-referenced by path) — worth it for anything non-trivial.
- Function level: what it does, why it exists, the parameter/return contract, and pointers to closely related functions or modules. Scale this to the function; a two-line helper doesn't need a preamble.
- Inline: complex logic, non-obvious decisions, must-know facts and gotchas, side effects (I/O, mutation, shared/global state, network), and `TODO:` notes for known future work.
- Comment the *why*, not the *what* — skip narration of lines that already read plainly. Comments that restate the code are worse than none.

**JavaScript / TypeScript**
- Prefer function expressions assigned to `const` (`const fn = () => {}`) over `function` declarations. Keeps binding and value in one place, blocks reassignment, avoids hoisting surprises, and reads consistently with the arrow callbacks already everywhere in JS/TS.
- Departures worth making: a `function` declaration where hoisting is genuinely wanted (mutual recursion, a helper used above its definition), generators, and anything needing its own `this` (object methods, non-arrow callbacks bound by a framework).

## Environment: Windows + WSL2

This machine is **Windows with WSL2 (Ubuntu-24.04)**. Claude Code runs inside WSL, but the GUI
applications — Chrome in particular — are installed on the **Windows** side. Do not assume a
Linux-only world.

### Browsers

**Do not install Linux Chrome/Chromium in WSL, and do not `apt-get install` its shared libraries
(`libnss3`, `libnspr4`, `libasound2t64`, …).** Chrome is already installed on Windows at:

```
/mnt/c/Program Files/Google/Chrome/Application/chrome.exe
```

Playwright's bundled Chromium under `~/.cache/ms-playwright/` in WSL is **non-functional** — it is
missing those system libs. `ldd` reporting "not found" there is expected, not a problem to fix.

### The WSL2 networking boundary (NAT mode)

Networking is NAT, not mirrored, so the two directions are **not** symmetric:

- **Windows → WSL works via `localhost`.** A dev server listening in WSL on port 5173 is reachable
  from Windows Chrome at `http://localhost:5173`. Bind with `--host` to be safe.
- **WSL → Windows requires the gateway IP**, not `localhost`. Get it with
  `ip route show default | awk '{print $3}'` (e.g. `172.x.x.1`).

### Driving Chrome for browser verification

Chrome **ignores `--remote-debugging-address=0.0.0.0`** and always binds `127.0.0.1:9222`, so a
CDP client running in WSL cannot reach it directly. Confirm with
`netstat.exe -ano | grep 9222` — it will show `127.0.0.1:9222`, never `0.0.0.0:9222`.

Workable approaches, in order of preference:

1. **Run the CDP client on Windows** (Windows `node.exe` / `python.exe`, both present), so
   everything stays on `127.0.0.1` and no boundary is crossed. WSL paths are reachable from
   Windows as `\\wsl.localhost\Ubuntu-24.04\...`.
2. **Relay the port**: a PowerShell TCP forwarder (PS 5 is available) from `0.0.0.0:9223` to
   `127.0.0.1:9222`. A raw TCP relay is fine — it passes both the HTTP and the WebSocket upgrade.
3. `netsh interface portproxy` — needs Administrator, usually unavailable.

Launching a fresh Chrome needs its own `--user-data-dir` (e.g. `C:\cdp-profile`); otherwise the
command just attaches to the already-running Chrome and prints "Opening in existing browser
session" instead of starting a debuggable instance.

### chrome-devtools-mcp

The MCP server fails here with `Target closed` because it tries to launch the broken Linux
Chromium. That is this environment problem, not a bug in the MCP server or in the app under test.
