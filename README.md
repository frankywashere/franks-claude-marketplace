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

## Installation

### Add the Marketplace

```bash
/plugin marketplace add <your-github-username>/franks-claude-marketplace
```

### Install a Plugin

```bash
/plugin install codex-exec
```

### Verify Installation

```bash
/plugin list
```

## Usage

Once installed, the plugins are automatically available in Claude Code. For example, the `codex-exec` skill will automatically trigger when you:

- Say "use codex" or "run codex"
- Ask for a second AI opinion on code
- Request specialized code generation or analysis

## Repository Structure

```
franks-claude-marketplace/
├── marketplace.json              # Catalog of all plugins
├── README.md                     # This file
└── codex-exec-plugin/           # Plugin directory
    ├── .claude-plugin/
    │   └── plugin.json          # Plugin manifest
    └── skills/
        └── codex-exec/
            └── SKILL.md         # Skill definition
```

## Adding More Plugins

To add additional plugins to this marketplace:

1. Create a new plugin directory (e.g., `my-new-plugin/`)
2. Add the plugin structure with `.claude-plugin/plugin.json` and your skill/command files
3. Update `marketplace.json` to include the new plugin
4. Commit and push changes

## Requirements

- Claude Code CLI installed
- For `codex-exec`: OpenAI Codex CLI installed at `~/.npm-global/bin/codex`

## License

MIT

## Author

Frank
