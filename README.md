# Nocturnal Agent

Nocutrnal is the cutting edge of coding agent and AI assistants, high performance and also highly capable, with a blazingly fast, class leading TUI and a feature complete lightweight Desktop app, as well as intergations for Telegram and Discord. Designed and written from the ground up in memory-safe Rust and compiled to native code. 

Free and multi provider, you can do all of your work with Nocturnal. Use any provider or model that you want, from local models to OpenAI, SpaceX AI, and other subscriptions.

Made with care by a performance and systems expert bringing to bear decades of experience making fast, light, powerful software. Nocturnal was used for its own creation and development.

## Highlights

| | |
|---|---|
| **Fast and lightweight** | Opens quickly, stays responsive, and uses minimal system resources. |
| **Works with your models** | Use OpenAI, Anthropic, Google, ChatGPT, local models, or compatible providers. |
| **Reliable provider fallback** | Automatically switches providers when one is unavailable. |
| **Agent-to-agent messaging** | Native agent-to-agent messaging. No hacks or filesystem polling. | 
| **Multi-agent workflows** | Delegate work to focused agents, persistent helpers, or parallel agent groups. |
| **Safe tool use** | Runs commands and file edits in a restricted environment separate from your secrets. |
| **Long-running sessions** | Pause, resume, schedule, and continue work across restarts. |
| **Review-gated work** | Require review before an agent marks important work complete. |
| **Terminal and desktop apps** | Use Nocturnal in the terminal or graphical desktop app. |
| **Multi-window** | Launch multiple TUIs and multiple desktop app windows. All communicate with the same backend. |
| **Mobile bridges** | Communicate via Telegram or Discord. |
| **Voice dictation** | Talk to Nocturnal with local push-to-talk transcription in both TUI and Desktop |
| **Large-context workflows** | Keep long projects manageable with compaction, memory, and persistent working state. |
| **Simple install** | Ships as a single command-line app without requiring Node, Python, or a database. |

## Install

### Linux, macOS, WSL2

For TUI and Desktop:

```bash
curl -fsSL https://nctrnl.dev/install | bash
```

For TUI only:

```bash
bash <(curl -fsSL https://nctrnl.dev/install) --tui-only
```

The installer will download and install the platform-appropriate binaries.

If you already have provider API keys set in your environment with compatible names, they will be picked up automatically.

Otherwise, run `nocturnal` (the TUI) and the enter an API key or connect an OAuth provider. Or run `nocturnal-desktop` and click the link to the providers settings page in banner that appears at the top of the window.

Don't forget to select a default model after having configured at least one provider.

When a new version is released you'll be prompted to upgrade.

See [Getting Started](docs/features/getting-started.md) for the full walkthrough, including verification, provider setup, and uninstall.

## Quickstart

```bash
nocturnal                  # launch the TUI in the current directory
nocturnal --resume <id>    # resume an existing session
```

In the TUI:

- `/models` — show every model from your configured providers
- `/connect` — add a provider interactively (persists to `~/.nocturnal/secrets/.env`)
- `/plugins telegram enable` (or `discord enable`) — wire up the Telegram or Discord bridge
- `/pause` and `/unpause` — halt and resume the session at the next tool boundary (survives restarts)

Sessions are scoped by working directory. Sub-agents sessions show up in the
panel below the chat.

See [Slash Commands](docs/features/slash-commands.md) for
the full list.

## Documentation

User documentation lives in [`docs/features/`](docs/features/) — one file per feature, each explaining what it does, how to configure it, and why it works the way it does.

### Core

- [Getting Started](docs/features/getting-started.md) — install, configure, first session
- [TUI](docs/features/tui.md) — the terminal interface, themes, key bindings
- [Desktop](docs/features/desktop.md) — the graphical client (Linux, macOS)
- [Sessions](docs/features/sessions.md) — session list, auto-titling, pinning
- [Client/Server](docs/features/client-server.md) — Unix socket IPC, standalone mode
- [CLI Mode](docs/features/cli-mode.md) — scripting and automation
- [Slash Commands](docs/features/slash-commands.md) — every `/command`

### Agents

- [Subagents](docs/features/subagents.md) — ephemeral, persistent, swarms, messaging
- [Agent Definitions](docs/features/agent-definitions.md) — YAML frontmatter, layering
- [Agent Skills](docs/features/agent-skills.md) — `SKILL.md` capability packages
- [Checklist Mode](docs/features/checklist-mode.md) — review-gated workflows
- [Heartbeat](docs/features/heartbeat.md) — periodic nudges
- [Cron](docs/features/cron.md) — scheduled jobs
- [Smart Edit](docs/features/smart-edit.md) — delegate edits to a cheap model

### Models and Providers

- [Providers](docs/features/providers.md) — OpenAI, Anthropic, Google, ChatGPT OAuth
- [Multi-Provider Aliases](docs/features/multi-provider-aliases.md) — failover and round-robin
- [Configuration Reference](docs/features/configuration-reference.md) — every config option

### Interaction

- [Streaming](docs/features/streaming.md) — chunk-level streaming, failover
- [Images](docs/features/images.md) — paste images, vision models, terminal rendering (ANSI or native graphics protocol)
- [Voice Dictation](docs/features/voice-dictation.md) — push-to-talk
- [External Editor](docs/features/external-editor.md) — click filenames, compose in `$EDITOR`

### Long-Running Workflows

- [Context Compaction](docs/features/context-compaction.md) — automatic summarization
- [Cross-Session Memory](docs/features/cross-session-memory.md) — `USER.md`, `MEMORY.md`
- [Pause and Resume](docs/features/pause-resume.md) — `/pause`, `/unpause`
- [Upgrade](docs/features/upgrade.md) — `/upgrade` and `/upgrade revert`
- [Loop Protection](docs/features/loop-protection.md) — stuck-loop mitigation
- [Session Storage](docs/features/session-storage.md) — JSONL files on disk

### Integrations

- [Telegram](docs/features/telegram.md)
- [Discord](docs/features/discord.md)
- [Plugins](docs/features/plugins.md) — built-in and external

### Safety and Internals

- [Architecture](docs/features/architecture.md) — components, IPC, state locations
- [Sandboxing](docs/features/sandboxing.md) — bwrap, sandbox-exec, write restrictions
- [Secrets (`.env`)](docs/features/dot-env.md) — `~/.nocturnal/secrets/.env`
- [Code Navigation](docs/features/code-navigation.md) — AST-based refactoring
- [Persistent REPL](docs/features/persistent-repl.md) — RLM Python REPL
- [Performance](docs/features/performance.md) — startup, memory, CPU numbers

## Performance

Measured on Linux x86_64 with the default configuration:

| Metric | Value |
|---|---|
| Startup (binary → ready) | ~13 ms |
| Memory (USS + swap) | ~22 MiB |
| CPU while streaming | 0.7-2 % |
| Background sessions | negligible CPU |

---

## Architecture (high level)

Nocturnal is shipped in native code, highly tuned for performance with a cutting edge TUI rendering pipeline. Its modular architecture allows for rapid development and stable operation. By default, all instances connect to a single backend/server. Tool execution occurs in a sandboxed context, offering security and peace of mind. No open network ports by default. 

---

## Download

Pre-built binaries are available for Linux x86_64, Linux aarch64, macOS arm64.

[GitHub Releases](https://github.com/nctrnlagnt/nocturnal/releases). 

Each archive ships with a detached `.sha256` checksum.

## License

Copyright (C) 2026 nctrnl.dev. All rights reserved.

## Project status

Nocturnal is in an open beta release. The current build is feature-complete for self-hosted use; You can help make Nocturnal better by providing bug reports and feature requests, which can be submitted on X.

Star the repo!

[Nocturnal Agent](https://x.com/nctrnlagnt)
