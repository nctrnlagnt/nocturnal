# Slash Commands

## What

Slash commands are typed directly in the TUI message input. They control session
behavior, manage plugins, and configure integrations without leaving the
conversation.

Type any command below in the input field and press Enter.

## How it works

Commands are intercepted before the message is sent to the agent. They execute
locally in the TUI or server — no LLM round-trip is needed.

## Commands

### Session lifecycle

#### /new

Start a brand-new session in the current directory.

#### /done

Mark the current session as completed. Completed sessions are hidden from the
session list. Aliased as `/complete`.

#### /pause

Pause the session at the next tool boundary. The agent finishes its current
tool call, then stops. The pause survives server restarts — the session stays
paused until you explicitly resume it.

#### /unpause

Resume a paused session. The agent picks up where it left off.

#### /title [title]

Set the session title. With an argument, sets a manual title. Without an
argument, generates a title from the current conversation context (no extra
LLM request). See [Sessions](sessions.md) for the auto-titling behavior.

#### /pin / /unpin

Pin or unpin the current session. Pinned sessions appear at the top of the
session list.

### Context and memory

#### /compact [args]

Compact the conversation context. Removes older messages and replaces them with
a summary, freeing up context window space. Runs automatically when the context
gets large, but you can trigger it manually at any time. See
[Context Compaction](context-compaction.md).

#### /btw \<question\>

Ask a lightweight question without full conversation context. Useful for quick
side queries ("btw what's the capital of Peru?") that don't need the full
session history. Saves tokens and avoids polluting the main conversation.

#### /debug-prompt

Show the system prompt for the current session. Useful for debugging what the
agent sees — including memory, tool definitions, and any injected context.

### Agents

#### /return

Ensure the result of an ephemeral session is delivered to its parent session.
Used in subagent workflows where the parent needs to receive the child's output.
See [Subagents](subagents.md).

### Providers and models

#### /models

Show the available models and providers configured in `config.yaml`. See
[Providers](providers.md).

#### /connect

Add providers interactively. Walks you through setting up a new LLM provider
(API key, base URL, model name) without editing `config.yaml` by hand. The
key is appended to `~/.nocturnal/secrets/.env` and the provider entry is
added to `~/.nocturnal/config.yaml`.

#### /reload

Reload `~/.nocturnal/config.yaml` and `~/.nocturnal/secrets/.env` without
restarting the server. Use this after editing API keys, provider settings,
or other configuration.

### Plugins and integrations

#### /plugins

Manage plugins. Subcommands:

- `list` — show all plugins and their status
- `enable <name>` — enable a plugin
- `disable <name>` — disable a plugin
- `reload <name>` — reload a plugin's configuration

See [Plugins](plugins.md) for how plugins are structured.

#### /heartbeat [on|off|minutes]

Enable or disable a periodic nudge for idle sessions. When on, the agent
receives a gentle prompt after a period of inactivity, keeping it engaged in
long-running tasks.

- `/heartbeat on` — enable with default interval
- `/heartbeat off` — disable
- `/heartbeat 15` — set interval to 15 minutes

See [Heartbeat](heartbeat.md).

#### /telegram

Configure the Telegram integration. See [Telegram](telegram.md).

#### /discord

Configure the Discord integration. See [Discord](discord.md).

#### /voice

Start voice dictation (push-to-talk transcription). See
[Voice Dictation](voice-dictation.md).

#### /images

Open the image picker. See [Images](images.md).

### Sandbox

#### /sandbox [on|off]

Toggle the sandbox for the current session. See
[Sandboxing](sandboxing.md#per-session-toggle) for details.

- `/sandbox` — show the current sandbox status as an out-of-band message
- `/sandbox on` — enable sandbox for this session
- `/sandbox off` — disable sandbox for this session

The toggle persists in `.meta.json` and is a no-op when the server-level
`sandbox.enabled` is `false`. Subagents inherit their root session's
effective state and cannot toggle their own.

### Maintenance

#### /upgrade [/revert]

Download and install the latest release. `/upgrade revert` restores the
previous binary and configuration. See [Upgrade](upgrade.md).

### Local TUI commands

These commands are handled locally by the TUI and do not reach the server.

| Command | Description |
|---------|-------------|
| `/export` | Export conversation to a timestamped file |
| `/share` | Copy the session ID to the clipboard |
| `/init` | Initialize the current project with a starter agent |
| `/status` | Show connection status |
| `/welcome-mode` | Cycle the welcome screen animation mode (`animated`, `static`, `none`) |
| `/movie-loop` | Toggle looping of the welcome animation |
| `/clear` | Clear the input field |
| `/help` | Show the local command help |

## Notes

- Commands are case-sensitive. Type them exactly as shown.
- Unrecognised commands (anything starting with `/` that isn't listed) are
  sent to the agent as a regular message.