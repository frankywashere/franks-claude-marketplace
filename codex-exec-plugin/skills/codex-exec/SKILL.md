---
name: codex-exec
description: Execute OpenAI Codex CLI for AI-powered code analysis, generation, refactoring, and reviews. Use when the user asks to "use codex", "run codex", "ask codex", or when you need a second AI opinion on code design, architecture decisions, or complex implementation strategies. Useful for delegating specialized tasks to another AI agent.
allowed-tools: Bash(codex:*)
---

# Codex CLI Agent Integration

This skill enables you to invoke OpenAI's Codex CLI as a specialized AI agent for code-related tasks.

## What is Codex?

Codex is OpenAI's standalone CLI tool (separate from Claude Code) that provides:
- AI-powered code generation and refactoring
- Codebase analysis and architecture review
- Automated code reviews
- Interactive and non-interactive execution modes
- Advanced reasoning with GPT-5.2-codex model

## When to Use This Skill

Use Codex when:
- User explicitly asks to "use codex" or "run codex"
- You want a second AI perspective on complex code decisions
- You need specialized code generation capabilities
- User requests automated code reviews
- You want to delegate a well-defined subtask to another AI agent

## Available Commands

### 1. Non-Interactive Execution (Primary Use)

```bash
codex exec "your prompt here"
```

**Common patterns:**

```bash
# Basic analysis
codex exec "Analyze the performance of train.py and suggest optimizations"

# Code generation - Codex writes files directly
codex exec "Generate unit tests for the UserService class and save to tests/test_user_service.py"

# With structured output
codex exec --json "List all API endpoints in this project"

# Fully automated (no approval prompts)
codex exec --full-auto "Refactor the authentication module for better security"

# Capture Codex's final RESPONSE to a file (NOT the files Codex creates!)
# WARNING: This saves Codex's closing message, not files it generates during execution
codex exec "Analyze the API" --output-last-message analysis_summary.txt

# Read from stdin
echo "Fix type errors in this file" | codex exec
```

**IMPORTANT: Understanding `--output-last-message`**

The `--output-last-message <file>` flag saves Codex's **final conversational response** to a file.
It does NOT save files that Codex creates during execution. If Codex creates a file called
`report.md` and you also use `--output-last-message report.md`, the response will OVERWRITE
the file Codex created!

- To have Codex create a file: Just ask it in the prompt, don't use `--output-last-message`
- To capture Codex's summary: Use `--output-last-message` with a DIFFERENT filename

### 2. Code Review

```bash
# Review uncommitted changes
codex review --uncommitted

# Review against base branch
codex review --base main

# Review specific commit
codex review --commit <SHA>

# With JSON output
codex review --uncommitted --json
```

### 3. Key Flags

| Flag | Purpose |
|------|---------|
| `--json` | Output structured JSONL for parsing |
| `--full-auto` | No approval prompts (workspace-write sandbox) |
| `--sandbox <MODE>` | `read-only`, `workspace-write`, or `danger-full-access` |
| `-m <MODEL>` | Override model (e.g., `gpt-5.2-codex`, `o3`) |
| `--output-last-message <FILE>` | Save final response to file |
| `-C <DIR>` | Set working directory |
| `--search` | Enable web search capability |
| `-c <KEY=VALUE>` | Override config (e.g., `-c model="o3"`) |

## Current Installation

**Codex Location**: `/Users/frank/.npm-global/bin/codex`
**Version**: 0.77.0
**Model**: gpt-5.2-codex (with xhigh reasoning effort)
**Config**: `~/.codex/config.toml`

Current user settings:
- Sandbox mode: workspace-write
- Approval mode: suggest
- Trusted directory: /Users/frank

## How to Invoke

### Example 1: Get Code Analysis
```bash
codex exec "What are the main performance bottlenecks in v7/training/trainer.py?"
```

### Example 2: Generate Code (Codex writes file directly)
```bash
# Codex will create the file itself - no --output-last-message needed
codex exec "Create a visualization dashboard for model metrics and save it to dashboard_metrics.py" \
  --sandbox workspace-write
```

### Example 3: Automated Review
```bash
codex review --uncommitted --json > review_results.jsonl
```

### Example 4: Multi-step Task with Full Automation
```bash
codex exec --full-auto "Add comprehensive error handling to the data pipeline and write tests"
```

## Instructions for You (Claude Code)

When invoking Codex:

1. **Choose the right mode**:
   - Use `codex exec` for one-shot tasks
   - Use `codex review` for PR/commit reviews
   - Add `--json` if you need to parse the output

2. **Set appropriate safety**:
   - `--sandbox read-only` for analysis only
   - `--sandbox workspace-write` for code changes (default)
   - `--full-auto` when user wants minimal friction

3. **Capture output**:
   - For file creation: Ask Codex to create the file in your prompt (no special flags needed)
   - For Codex's response: Use `--output-last-message <file>` BUT use a DIFFERENT filename than any file Codex creates
   - Use `--json` for structured parsing
   - WARNING: `--output-last-message` saves Codex's final chat message, NOT files it creates during execution

4. **Present results**:
   - Show the command you ran
   - Display Codex's response
   - Offer to apply any suggested changes

5. **Handle errors**:
   - If codex fails, check that it's installed: `which codex`
   - Verify the prompt is clear and specific
   - Consider adjusting sandbox/approval settings

## Important Notes

- Codex runs in its own execution context with its own tools
- It can read/write files directly (based on sandbox settings)
- Output is streamed in real-time (or as JSONL with --json)
- Sessions are logged in `~/.codex/sessions/`
- Codex has access to web search with `--search` flag

## Example Workflow

**User**: "Use codex to analyze our training pipeline performance"

**You should**:
1. Run: `codex exec "Analyze the performance characteristics of the training pipeline in v7/training/trainer.py, identify bottlenecks, and suggest optimizations"`
2. Show the command you ran
3. Display Codex's analysis
4. Offer to implement suggested optimizations

## Troubleshooting

- **"stdin is not a terminal"**: You forgot `exec` - use `codex exec` not just `codex`
- **Command hangs**: Add timeout: `timeout 300 codex exec "..."`
- **Permission denied**: Check sandbox mode or use `--full-auto`
- **Not installed**: Codex is at `/Users/frank/.npm-global/bin/codex`

## References

For detailed Codex documentation, see:
- Official docs: https://developers.openai.com/codex/cli/
- Command reference: https://developers.openai.com/codex/cli/reference/
- GitHub: https://github.com/openai/codex
