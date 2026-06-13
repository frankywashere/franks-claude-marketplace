---
name: gemini
description: Use the locally installed Gemini CLI (Google) headlessly to delegate code-related work to another AI agent. Trigger when the user says "use gemini", "run gemini", "ask gemini", "have Gemini …", "get Gemini's take", or wants a second AI opinion on code design, architecture, a diagnosis, an alternate implementation, or a code review. Invokes `gemini -p` via Bash and relays the result. Do NOT use for other CLI agents (Codex, Grok) or for work the main Claude thread should just do itself.
allowed-tools:
  - Bash(gemini *)
  - Bash(gemini)
  - Read
---

# Gemini CLI (headless) skill

Drive the locally installed **Gemini CLI** (`gemini`, by Google) as an external agent from inside
Claude Code. Gemini is a Claude-Code-style agentic CLI; its **non-interactive headless mode** (`-p`)
prints a result to stdout and exits, which is what makes it scriptable from Bash.

The binary is at `/Users/frank/.npm-global/bin/gemini` (also `/usr/local/bin/gemini`). Verified with
**v0.29.0**. Auth on this machine is **OAuth personal**, cached in `~/.gemini/oauth_creds.json` — if a
call fails with an auth error, the user must run `gemini` once interactively and pick a login (it's
interactive; suggest they type `! gemini`). No API key is required for the cached-OAuth path.

## Core invocation

```bash
gemini -p "<TASK PROMPT>" -o json 2>/tmp/gemini.err >/tmp/gemini.json
```

- **`-p, --prompt "<prompt>"`** — non-interactive (headless). Prints response to stdout, exits. (Required.)
  Anything piped on **stdin is appended** to this prompt (verified), which is handy for feeding code/errors.
- **`-o, --output-format {text,json,stream-json}`** — use `json` to parse reliably; `text` (default) when
  you just want to relay prose to the user.
- **Redirect stderr separately** (`2>...err >...json`). Gemini writes `Loaded cached credentials`,
  quota-retry notices, and (on failure) the error report to **stderr**; the clean result goes to **stdout**.
- **`-m, --model <id>`** — pin a model. Default is **`gemini-3-flash-preview`** (fast, reliable on the free
  OAuth tier). `gemini-3-pro-preview` is the valid Pro name but is **quota-limited** on this account (see
  Quota below) — only reach for it on genuinely hard problems and expect possible `exhausted capacity` errors.
- **`--include-directories <a,b>`** — add extra dirs to the workspace context (comma-separated or repeated).
- Run from / against the project root so Gemini reads the right repo (it uses the shell cwd; there is no
  `--cwd` flag — `cd` into the target or pass `--include-directories`).

## Read-only vs. write-capable (verified behavior)

Gemini's `--approval-mode` governs whether it may run tools headlessly. **Read tools auto-run even in
default mode**, so review/analysis needs no special flag.

- **Read-only / Q&A / review / diagnosis (safe default):** no approval flag. Verified: in default mode a
  headless run will auto-execute *read* tools (it read `marketplace.json` and answered) but will not edit.
  Optionally also say "read-only — do not modify files" in the prompt.
- **Write-capable (let Gemini edit files):** add **`--approval-mode auto_edit`** — auto-approves edit/write
  tools. **Verified working** (created a file headlessly). Only when the user clearly asked Gemini to make
  changes. Prefer running it inside a throwaway git branch/worktree so edits are easy to review/revert.
- **`--yolo` / `--approval-mode yolo`** (auto-approve *all* tools incl. shell): **blocked on this machine by
  an org policy** (`secureModeEnabled`) — it exits non-zero (`52`) with `YOLO mode is disabled by your
  administrator`. **Do not use it here; use `auto_edit` for writes instead.**
- **`--approval-mode plan`** (read-only planning mode) requires `experimental.plan` to be enabled in
  settings; otherwise it hard-errors (`Approval mode "plan" is only available when experimental.plan is
  enabled`). Don't pass it unless that flag is on — plain default mode already covers read-only.

| `--approval-mode` | Effect | Use it for |
|---|---|---|
| `default` (omit flag) | Prompts for tool approval; reads still auto-run headlessly | Review, Q&A, diagnosis |
| `auto_edit` | Auto-approves edit/write tools | Letting Gemini make code changes |
| `yolo` | Auto-approves all tools — **blocked by org policy here** | (unavailable) |
| `plan` | Read-only planning — needs `experimental.plan` | (skip unless enabled) |

Also useful: **`--allowed-tools <a,b>`** to allowlist specific tools that may run without confirmation,
and **`--allowed-mcp-server-names <...>`** to restrict MCP servers.

## Session continuity

- **`-r, --resume latest`** — resume the most recent session for this project. **`--resume <index>`**
  resumes a numbered one. Verified: a resumed run recalled state from the prior session.
- **`--list-sessions`** — list sessions for the current project (with index + id). **`--delete-session
  <index>`** removes one. Sessions are scoped per project directory.

Use these when the user says "ask Gemini to keep going / follow up / refine its last answer".

## JSON output schema

On **success**, stdout is a single JSON object (exit `0`):

```json
{
  "session_id": "734abe34-b9a2-4fba-9f7c-396ade96970f",
  "response": "<assistant reply text>",
  "stats": {
    "models": { "gemini-3-flash-preview": { "api": { "totalRequests": 1, ... }, "tokens": { ... } } },
    "tools": { "totalCalls": 1, "totalSuccess": 1, ... },
    "files": { "totalLinesAdded": 0, "totalLinesRemoved": 0 }
  }
}
```

- **`.response`** is the reply — extract with `jq -r '.response'`.
- **`.session_id`** feeds `--resume` (by index via `--list-sessions`).
- **`.stats.tools.totalCalls`** / **`.stats.files`** tell you whether Gemini ran tools or edited files.

## Failure detection — exit code is reliable (unlike Grok)

Gemini **exits non-zero on failure** and writes the error to **stderr** while leaving **stdout empty**.
So check the exit code first; the error JSON `{"session_id","error":{"type","message","code"}}` also lands
on stderr, but its `.message` is often unhelpful (`[object Object]`) — the human-readable cause is in the
stderr prose above it (e.g. `ModelNotFoundError`, `RetryableQuotaError`, `YOLO mode is disabled…`).

```bash
gemini -p "$PROMPT" -o json 2>/tmp/gemini.err >/tmp/gemini.json
if [ $? -ne 0 ] || [ ! -s /tmp/gemini.json ]; then
  echo "GEMINI FAILED:"; grep -iE 'error|exhausted|not found|disabled|denied' /tmp/gemini.err | head -5
else
  jq -r '.response' /tmp/gemini.json   # success
fi
```

Do **not** fake success — surface the real stderr cause. (Common exit codes seen: `1` API/model error,
`52` org-policy block.)

## Quota & retries (free OAuth tier)

The free OAuth-personal tier is rate-limited. On a hit you'll see `You have exhausted your capacity on
this model. Your quota will reset after Ns` on stderr; Gemini **auto-retries with backoff**. Default
`gemini-3-flash-preview` usually recovers after a short retry. `gemini-3-pro-preview` can hit a hard wall
(`Max attempts reached`) and then fail with a non-zero exit. Prefer flash for routine work; treat a pro
quota error as a real failure and fall back to flash or report it.

## How to run it (operationally)

1. Compose a tight, self-contained prompt — Gemini does **not** share Claude's conversation context, so
   inline all background, absolute file paths, pasted code/errors, and the exact ask. State explicitly
   whether Gemini may edit files or must stay read-only.
2. For a read-only second opinion/review, no approval flag. For edits, add `--approval-mode auto_edit`.
3. Cold start + model run can take tens of seconds, and quota backoff adds more. **Run the Bash call with
   `run_in_background: true`** and poll the output file rather than blocking the foreground tool timeout.
4. Parse the result (schema above), check the exit code, then relay. **Attribute the answer to Gemini**,
   and if it edited files (write mode) say so and summarize what changed (`.stats.files`).

## Notes & gotchas

- **No `--cwd`.** Gemini runs against the shell's current directory. `cd` into the project root (or use
  `--include-directories`) so it reads the right code.
- **MCP is separate from Claude's.** Gemini has its own MCP config (`gemini mcp list/add/...`); it does not
  inherit Claude Code's servers. So you won't see Claude's MCP startup noise here.
- **Org-managed policy.** `secureModeEnabled` (an admin policy, not a local file) disables YOLO/full-auto on
  this account — don't try to "fix" it; just use `auto_edit`. Policy info: https://goo.gle/manage-gemini-cli.
- Other surfaces exist but are out of scope for headless delegation: `gemini extensions`, `gemini skills`,
  `gemini hooks` (incl. `gemini hooks migrate` to import Claude Code hooks). Mention them only if asked.

## References

- Repo / docs: https://github.com/google-gemini/gemini-cli
- Run `gemini --help` for the authoritative, version-specific flag list (this skill was verified on v0.29.0).
