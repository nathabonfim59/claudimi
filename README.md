# Claudimi

<img width="761" height="558" alt="image" src="https://github.com/user-attachments/assets/50989d79-efa4-499a-899b-5afe5ce1b8bc" />

A wrapper script that runs [Claude Code](https://docs.anthropic.com/en/docs/claude-code) with [Kimi](https://www.kimi.com) as the backend, via Kimi's Anthropic-compatible API.

Kimi Code benefits include Claude Code access through Kimi's endpoint. This wrapper makes `claude-kimi` a drop-in replacement for `claude`, including spawning teammates for parallel work.

## What it does

`claude-kimi` is a shell wrapper around the official `claude` CLI that:

- Points the Anthropic SDK at Kimi's API (`https://api.kimi.ai/coding/`)
- Maps Kimi models to Claude model tiers so existing prompts and tooling work unchanged
- Isolates all configuration under `~/.claudimi` instead of `~/.claude`

## Model mapping

The top tiers run K3 with a 1M-token context window, each with its own default thinking effort. Sonnet runs the same K3 pinned to 256k context at low thinking; Haiku runs `kimi-for-coding` (upgraded in place to K2.8 Preview, still 1M context):

| Claude tier | Kimi model       | Default effort | Kimi thinking |
|-------------|------------------|----------------|---------------|
| Opus        | k3 (1M)          | `high`         | high          |
| Sonnet      | k3-256k          | `low`          | low           |
| Haiku       | kimi-for-coding  | `low`          | low           |
| Fable       | k3 (1M)          | `xhigh`        | max           |

`kimi-for-coding-highspeed` (the fast variant, 256k context, Pro+ plans only) is also selectable in the `/model` picker as **Kimi Highspeed**.

**Sonnet's 256k pin:** the wrapper's context variables stay at 1M for the other tiers, so Claude Code assumes Sonnet has 1M as well. If Kimi enforces the pin, long Sonnet sessions may error past 256k; use Opus for long sessions.

### How effort control works

- Kimi converts Claude Code's effort values to K3 thinking levels: `low` becomes low, `medium` and `high` become high, `xhigh` and `max` become max. The wrapper sets `CLAUDE_CODE_ALWAYS_ENABLE_EFFORT=1` because Claude Code doesn't recognize Kimi model IDs as effort-capable and would skip the parameter without it. Per-tier defaults live in `modelSettings` in `settings.json`.
- The three K3 tiers need three distinct `modelSettings` keys to carry three effort levels. Claude Code strips the `[1m]` suffix before sending (it's only a client-side context-window hint), so `k3` and `k3[1m]` reach Kimi as the same model but stay distinct keys. Fable uses the bare ID, and `CLAUDE_CODE_MAX_CONTEXT_TOKENS=1048576` keeps it on the 1M window; Sonnet uses the `k3-256k` pin as its third key.
- `/effort` overrides the tier default for the current model in-session; confirming with <kbd>Enter</kbd> saves the choice back into `modelSettings`.

### Auto mode classifier checks

Auto mode's no-charge classifier checks run on Anthropic's servers, so they can't work through Kimi's gateway. claudimi sets `CLAUDE_CODE_AUTO_MODE_SERVER=0` to stop Claude Code from asking for them, which also suppresses the "this session isn't eligible" notice. Auto mode keeps working: classifier checks run as regular model requests billed to your Kimi usage, never to an Anthropic subscription, since every request routes through the Kimi key.

## Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) installed (`claude` in `$PATH`)
- A `KIMI_API_KEY` environment variable set with your Kimi Code API key

## Quick start

```bash
curl -fsSL https://raw.githubusercontent.com/nathabonfim59/claudimi/main/install.sh | bash
```

Re-running the same command updates an existing installation in place: it checks the latest release and only refreshes the wrapper and skill when a newer version is available. From inside a claudimi session, the `/claude-kimi-update` command does the same thing.

The installer:

1. Sets your `KIMI_API_KEY` (saved to `~/.bashrc` or `~/.zshrc`)
2. Downloads `claude-kimi` to `~/.local/bin`
3. Copies the recommended `settings.json` to `~/.claudimi/`
4. Installs the teammate skill via `npx`

All Claude Code flags and arguments are passed through to `claude` unchanged.

## Versioning & updates

claudimi is versioned with `vX.Y.Z` [GitHub releases](https://github.com/nathabonfim59/claudimi/releases). The installed version is printed by:

```bash
claude-kimi --version     # or: claude-kimi -V
```

The updater compares your installed version against the latest release and only downloads when a newer version exists; if you're already current it prints `Already up to date` and does nothing. Two flags are available when piping the installer to bash:

```bash
# Just report installed vs. latest, then exit
curl -fsSL https://raw.githubusercontent.com/nathabonfim59/claudimi/main/install.sh | bash -s -- --check

# Force a refresh even when already up to date
curl -fsSL https://raw.githubusercontent.com/nathabonfim59/claudimi/main/install.sh | bash -s -- --force
```

If the latest version can't be determined (no release found yet, or a network error), the updater falls back to refreshing the files rather than blocking the update.

### Cutting a release (maintainers)

1. Bump `CLAUDE_KIMI_VERSION` in [`claude-kimi`](claude-kimi) — it's the single source of truth.
2. Commit and push to `main`.
3. Tag and push: `git tag vX.Y.Z && git push origin vX.Y.Z`.

A [GitHub Action](.github/workflows/release.yml) then verifies the tag matches the version (failing the run on mismatch) and publishes the release. Once published, `--check` and every updater run pick up the new version.

## Configuration directory: `~/.claudimi`

This wrapper sets `CLAUDE_CONFIG_DIR` to `~/.claudimi`, so all Claude Code state lives there instead of the default `~/.claude`:

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

Any configuration you'd normally put in `~/.claude` goes in `~/.claudimi` instead:

- **Settings** - edit `~/.claudimi/settings.json` (or use `/config` inside the session - it writes to the same place)
- **Status line** - set the `statusLine` key in `~/.claudimi/settings.json`
- **Per-project settings** - go under `~/.claudimi/projects/`
- **Memory files** - stored under `~/.claudimi/projects/<project>/memory/`

The in-app UI (settings panels, `/config`, etc.) works the same; it reads and writes to `~/.claudimi` behind the scenes.

## Status line

The included `settings.json` configures [cc-statusline](https://github.com/nathabonfim59/cc-statusline), a status line that shows context usage, cost, timing, git state, and diff stats. It also helps when using teammates: a `tmux capture-pane` snapshot reveals the teammate's context fill level and whether it has uncommitted changes.

Install it:

```bash
curl -fsSL https://raw.githubusercontent.com/nathabonfim59/cc-statusline/main/install.sh | bash
```

See the [cc-statusline repo](https://github.com/nathabonfim59/cc-statusline) for theming, custom layouts, and other options.

## Teammate skill

The [`skills/claude-kimi-teammate/`](skills/claude-kimi-teammate/) directory contains a Claude Code skill that spawns `claude-kimi` instances as interactive teammates in tmux. This recreates the built-in teammate feature on Kimi's API.

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

Once installed, Claude Code picks it up automatically and can spawn teammates when asked to delegate work.

## Why a separate config dir?

Keeping `~/.claudimi` separate from `~/.claude` means your real Claude Code setup and your Kimi setup don't interfere with each other. You can run either one independently with its own history, sessions, and settings.

Memories are not shared between the two: anything saved via `/remember` or the memory system in your regular Claude Code setup won't be visible inside `claude-kimi`, and vice versa.

To share memories or other state, symlink specific folders. For example, to share project memories:

```bash
ln -s ~/.claude/projects ~/.claudimi/projects
```

## License

MIT
