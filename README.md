# Frank's Claude Code Marketplace

A personal collection of Claude Code plugins featuring AI agent integrations and productivity tools.

## Available Plugins

### codex-exec

Execute OpenAI Codex CLI for AI-powered code analysis, generation, refactoring, and reviews. Enables Claude Code to delegate specialized tasks to another AI agent.

**Features:**
- AI-powered code generation and refactoring
- Codebase analysis and architecture review
- Automated code reviews
- Interactive and non-interactive execution modes
- Advanced reasoning with GPT-5.2-codex model

**Version:** 1.0.0

### grok-cli

Drive the **Grok CLI** (xAI) as a headless agent from Claude Code. Delegates tasks to `grok -p` — second opinions, root-cause diagnosis, alternate implementations, or substantial coding work — and relays the result back.

**Features:**
- A `grok` skill for direct, on-demand calls ("ask Grok to …")
- A `grok-rescue` subagent for autonomous multi-step hand-offs
- Headless single-turn invocation with JSON output parsing
- Read-only and write-capable (worktree-isolated) modes
- **Image & video generation** via Grok Imagine (`/imagine`, `/imagine-video`) — works headlessly
- Session continuity (`--continue` / `--resume`), `--best-of-n`, and self-verify (`--check`)
- Robust failure detection (parses output, not exit code) and documented MCP/effort-flag gotchas

**Version:** 1.2.0

## Installation

### Add the Marketplace

```bash
/plugin marketplace add frankywashere/franks-claude-marketplace
```

### Install a Plugin

```bash
/plugin install codex-exec
/plugin install grok-cli
```

### Verify Installation

```bash
/plugin list
```

## Usage

Once installed, the plugins are automatically available in Claude Code.

- **codex-exec** triggers when you say "use codex" / "run codex", ask for a second AI opinion on code, or request specialized code generation or analysis.
- **grok-cli** triggers when you say "ask Grok", "have Grok …", or "get Grok's take". For heavier multi-step hand-offs, the `grok-rescue` subagent takes over.

## Repository Structure

```
franks-claude-marketplace/
├── .claude-plugin/
│   └── marketplace.json          # Catalog of all plugins
├── README.md                     # This file
├── codex-exec-plugin/            # Plugin directory
│   ├── .claude-plugin/
│   │   └── plugin.json           # Plugin manifest
│   └── skills/
│       └── codex-exec/
│           └── SKILL.md          # Skill definition
└── grok-cli-plugin/              # Plugin directory
    ├── .claude-plugin/
    │   └── plugin.json           # Plugin manifest
    ├── skills/
    │   └── grok/
    │       └── SKILL.md          # Skill definition
    └── agents/
        └── grok-rescue.md        # Subagent definition
```

## Adding More Plugins

To add additional plugins to this marketplace:

1. Create a new plugin directory (e.g., `my-new-plugin/`)
2. Add the plugin structure with `.claude-plugin/plugin.json` and your skill/command/agent files
3. Update `.claude-plugin/marketplace.json` to include the new plugin
4. Commit and push changes

## Requirements

- Claude Code CLI installed
- For `codex-exec`: OpenAI Codex CLI installed at `~/.npm-global/bin/codex`
- For `grok-cli`: Grok CLI installed (`~/.grok/bin/grok`) and authenticated (`grok login`)

## License

MIT

## Author

Frank
