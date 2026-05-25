---
name: grok
description: Delegate a task to the Grok CLI agent (xAI) running headlessly in the terminal. Use when the user says "ask Grok", "have Grok do X", "get Grok's take", wants a second opinion / alternate implementation from Grok, or wants a task run through Grok. Invokes `grok -p` non-interactively via Bash and relays the result. Do NOT use for other CLI agents (e.g. Codex) or for work the main Claude thread should just do itself.
allowed-tools:
  - Bash(grok *)
  - Bash(grok)
  - Read
---

# Grok CLI (headless) skill

Drive the locally installed **Grok CLI** (`grok`, by xAI) as an external agent from inside
Claude Code. Grok is a Claude-Code-style agentic CLI; its **single-turn headless mode** prints a
result to stdout and exits, which is what makes it scriptable from Bash.

The binary is typically at `~/.grok/bin/grok` (also symlinked as `agent`). Auth is cached in
`~/.grok/auth.json` — if a call fails with an auth error, the user must run `grok login`
themselves (it's interactive; suggest they type `! grok login`).

## Core invocation

```bash
grok -p "<TASK PROMPT>" --output-format json --cwd "$PWD" 2>/tmp/grok.err >/tmp/grok.json
```

- **`-p, --single "<prompt>"`** — single-turn headless. Prints response to stdout, exits. (Required.)
- **`--output-format {plain,json,streaming-json}`** — use `json` when you need to parse the result
  reliably; use `plain` when you just want to relay prose to the user.
- **`--cwd <path>`** — run against a specific directory (default: current dir). Pass the project
  root explicitly so Grok reads the right repo.
- **Redirect stderr separately** (`2>...err >...json`). Grok writes logs / MCP errors to **stderr**
  and the clean result to **stdout**, so this keeps the JSON parseable.
- **`-m, --model <id>`** — pin a model. Leave unset to use the default. `grok models` lists what
  your account exposes.

> **⚠ Check `grok models` before using `--effort` / `--reasoning-effort`.** These flags appear in
> `--help`, but non-reasoning models reject the reasoning-effort parameter and the call hard-fails
> with `400: Model <id> does not support parameter reasoningEffort`. If your account only exposes a
> build/non-reasoning model (e.g. `grok-build`), **omit the effort flags entirely.** Only pass them
> when `grok models` shows a reasoning-capable model.

## Read-only vs. write-capable

Grok shares Claude Code's permission model and reads the repo's `.claude/settings*.json`. By
default a project may be **untrusted**, so Grok won't freely run tools/edits headlessly.

- **Read-only / Q&A / review / diagnosis (safe default):** no extra flags. Optionally harden with
  `--disable-web-search` and a restricted tool set. Also say "read-only — do not edit" in the prompt.
- **Write-capable (let Grok edit files / run commands):** add **`--always-approve`** to auto-approve
  tool executions, or set **`--permission-mode acceptEdits`** (modes: `default, acceptEdits, auto,
  dontAsk, bypassPermissions, plan`). Only when the user clearly asked Grok to make changes. Prefer
  isolating in a git worktree: add **`-w` / `--worktree [name]`**.
- Restrict the toolset with **`--tools <csv>`** (allowlist) / **`--disallowed-tools <csv>`** (denylist).

## Session continuity

- **`-c, --continue`** — continue the most recent Grok session for the current cwd.
- **`-r, --resume [SESSION_ID]`** — resume a specific session (the `.sessionId` from a prior JSON
  result), or the latest if omitted.
- `grok sessions` lists sessions; `grok export` dumps a transcript as Markdown.

Use these when the user says "ask Grok to keep going / follow up / refine its last answer".

## Power flags (headless-only)

- **`--best-of-n <N>`** — run the task N ways in parallel, return the best. Great for hard one-shot
  problems; costs N× the tokens, so only with user intent or for genuinely hard tasks.
- **`--check`** — append a self-verification loop (Grok double-checks its own work).
- **`--max-turns <N>`** — cap agent turns to bound cost on open-ended tasks.
- **`--prompt-file <path>`** — feed a long prompt from a file (avoids shell-quoting pain).
  `--verbatim` sends the prompt exactly as given.
- **`--rules "<text>"`** / **`--system-prompt-override "<text>"`** — inject extra rules / replace
  the system prompt.

## How to run it (operationally)

1. Compose a tight, self-contained prompt — Grok does **not** share Claude's conversation context,
   so inline all background, absolute file paths, and the exact ask.
2. For big prompts, write to a temp file and use `--prompt-file` to dodge quoting issues.
3. Cold start is slow (model spin-up, plus any MCP connect timeouts — see below), often tens of
   seconds to minutes before output, and a real task can run several minutes. **Run the Bash call
   with `run_in_background: true`** and poll the output file rather than blocking the foreground
   tool timeout.
4. Parse the result (see schema below) before relaying. Attribute the answer to Grok, and flag if
   Grok made file changes.

## JSON output schema

On success, stdout is a single JSON object:

```json
{
  "text": "<assistant reply>",
  "stopReason": "EndTurn",
  "sessionId": "019e6145-5cf9-7a80-9918-d8f3327416bc",
  "requestId": "65069e8a-082a-46e3-96c8-4908748f9e49",
  "thought": "...reasoning trace..."
}
```

- **`.text`** is the reply — extract with `jq -r '.text'`.
- **`.sessionId`** feeds `--resume <id>` to continue this exact session.
- **`.stopReason`** — e.g. `EndTurn` (normal); watch for other values on truncated runs.

On failure, stdout is instead `{"type":"error","message":"..."}` (and a top-level `Error:` line may
print). **Detect failure by parsing, not exit code** — Grok exits `0` even on API errors:

```bash
grok -p "$PROMPT" --output-format json --cwd "$PWD" 2>/tmp/grok.err >/tmp/grok.json
if jq -e '.type == "error"' /tmp/grok.json >/dev/null 2>&1 || ! jq -e 'has("text")' /tmp/grok.json >/dev/null 2>&1; then
  echo "GROK FAILED:"; jq -r '.message // .' /tmp/grok.json   # report loudly, do not fake success
else
  jq -r '.text' /tmp/grok.json
fi
```

## Inherited MCP servers — expect possible stderr errors (non-fatal)

Grok merges **Claude Code's MCP servers** from `~/.claude.json`. If any of those servers is
unreachable (e.g. a LAN/remote server that's down or off-network), Grok logs a non-fatal error to
**stderr** like:

```
ERROR worker quit with fatal: Transport channel closed ... <host:port> ... Host is down / No route to host
```

Grok logs it and continues — the real result still lands on stdout — but it adds startup latency
(the connect timeout). Because it's on stderr, redirecting stderr separately (as above) keeps stdout
clean. Run **`grok mcp doctor`** to diagnose MCP connectivity, and `grok mcp list` to see Grok's own
(separately configured) servers. Don't edit the user's global config to "fix" this unless asked.
