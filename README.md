# Claudimi

A wrapper script that runs [Claude Code](https://docs.anthropic.com/en/docs/claude-code) with [Kimi](https://www.kimi.com) as the backend provider, using Kimi's Anthropic-compatible API.

**Why?** Kimi Code benefits let you use the same Claude Code experience through Kimi's endpoint. This wrapper lets you use it as a drop-in replacement, including spawning teammates for parallel work.

## What it does

`claude-kimi` is a thin shell wrapper around the official `claude` CLI that:

- Points the Anthropic SDK at Kimi's API (`https://api.kimi.com/coding/`)
- Uses your `KIMI_API_KEY` as the Anthropic API key
- Isolates all configuration under `~/.claudimi` instead of `~/.claude`

Kimi's endpoint handles model routing automatically, so existing Claude Code prompts and model selectors keep working unchanged.

## Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) installed (`claude` in `$PATH`)
- A `KIMI_API_KEY` environment variable set with your Kimi Code API key

## Quick start

```bash
curl -fsSL https://raw.githubusercontent.com/nathabonfim59/claudimi/main/install.sh | sh
```

The installer will walk you through:

1. Setting your `KIMI_API_KEY` (saved to `~/.bashrc` or `~/.zshrc`)
2. Downloading `claude-kimi` to `~/.local/bin`
3. Copying the recommended `settings.json` to `~/.claudimi/`
4. Installing the teammate skill via `npx`

All Claude Code flags and arguments are passed through to `claude` unchanged.

## Configuration directory: `~/.claudimi`

This wrapper sets `CLAUDE_CONFIG_DIR` to `~/.claudimi`, which means **all** Claude Code state lives there instead of the default `~/.claude`:

```
~/.claudimi/
├── settings.json        # Global settings (model, status line, env vars, etc.)
├── .claude.json         # Internal state
├── history.jsonl        # Conversation history
├── projects/            # Per-project settings and memory
├── sessions/            # Session data
├── plans/               # Saved plans
└── ...
```

**This is important:** any configuration you'd normally put in `~/.claude` goes in `~/.claudimi` instead. For example:

- **Settings** - edit `~/.claudimi/settings.json` (or use `/config` inside the session - it writes to the same place)
- **Status line** - set the `statusLine` key in `~/.claudimi/settings.json`
- **Per-project settings** - go under `~/.claudimi/projects/`
- **Memory files** - stored under `~/.claudimi/projects/<project>/memory/`

The in-app UI (settings panels, `/config`, etc.) works the same - it just reads and writes to `~/.claudimi` behind the scenes.

## Status line

The included `settings.json` already configures [cc-statusline](https://github.com/nathabonfim59/cc-statusline) - a fast, themeable status line that shows context usage, cost, timing, git state, and diff stats. It also helps when using teammates: a `tmux capture-pane` snapshot reveals the teammate's context fill level and whether it has uncommitted changes.

Just install it:

```bash
curl -fsSL https://raw.githubusercontent.com/nathabonfim59/cc-statusline/main/install.sh | sh
```

See the [cc-statusline repo](https://github.com/nathabonfim59/cc-statusline) for theming, custom layouts, and other options.

## Teammate skill

The [`skills/claude-kimi-teammate/`](skills/claude-kimi-teammate/) directory contains a Claude Code skill that spawns `claude-kimi` instances as interactive teammates in tmux. This recreates the built-in teammate feature but using Kimi's API instead, so you get the same multi-agent workflow.

### How it works

- Spawns a new tmux window running `claude-kimi --dangerously-skip-permissions`
- Communicates between the orchestrator and teammates via `tmux send-keys`
- Teammates message the orchestrator by typing into its pane
- You can watch and steer any teammate by attaching to the tmux session

### Prerequisites

- [tmux](https://github.com/tmux/tmux) installed and your main Claude Code session running inside it
- The skill files placed in your project's `.claude/skills/` directory

### Install the skill

**Option 1: via npx (recommended)**

```bash
npx skills add nathabonfim59/claudimi -a claude-code -g -y
```

**Option 2: clone the repo**

```bash
git clone https://github.com/nathabonfim59/claudimi.git
```

Then copy `skills/claude-kimi-teammate/` into your project's `.claude/skills/`.

Once installed, Claude Code will pick it up automatically and can spawn teammates when asked to delegate work.

## Why a separate config dir?

Keeping `~/.claudimi` separate from `~/.claude` means your real Claude Code setup and your Kimi setup don't interfere with each other. You can run either one independently with its own history, sessions, and settings.

This also means **memories are not shared** between the two. Anything you saved via `/remember` or the memory system in your regular Claude Code setup won't be visible inside `claude-kimi`, and vice versa.

If you want to share memories (or other state) between the two, you can symlink specific folders. For example, to share project memories:

```bash
ln -s ~/.claude/projects ~/.claudimi/projects
```

## License

MIT
