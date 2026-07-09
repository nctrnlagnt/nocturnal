# Architecture

Nocturnal is structured as a small set of focused components that
communicate over well-defined channels. The shapes are deliberately
simple so the system is easy to reason about and easy to run anywhere
— from a `$5` VPS to a GPU workstation.

## The components

```
┌────────────────────────────────────────────────────────────────┐
│                       Your machine                              │
│                                                                 │
│   ┌──────────┐    Unix socket / HTTP      ┌──────────────────┐  │
│   │   TUI    │ ◄──────────────────────► │   LLM server     │   │
│   │ (client) │    (bearer auth)           │  (sessions,      │   │
│   └──────────┘                           │   providers,     │   │
│   ┌──────────┐    Unix socket / HTTP      │   agent defs,    │   │
│   │ Desktop  │ ◄──────────────────────► │   event bus)     │   │
│   │ (client) │    (bearer auth)           │                  │   │
│   └──────────┘                           └──────────────────┘   │
│         │                                          │               │
│         │                                Sandboxed │               │
│         │                                  process │               │
│         │                                          ▼               │
│         │                                ┌──────────────┐       │
│         │                                │ Tool executor│       │
│         │                                │ (bwrap /     │       │
│         │                                │  sandbox-exec)│      │
│         │                                └──────────────┘       │
│         ▼                                                       │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐                     │
│   │ Telegram │  │ Discord  │  │  Voice   │   Optional daemons  │
│   │ bridge   │  │ bridge   │  │ dictation│                     │
│   └──────────┘  └──────────┘  └──────────┘                     │
└────────────────────────────────────────────────────────────────┘
```

### LLM server

The brain. Owns sessions, credentials, provider connections, agent
definitions, the event bus, the cron scheduler, and the heartbeat
timers. Every conversation lives here. Sessions are persisted as
append-only JSONL files on disk; nothing is held in memory beyond what
is needed for the active session.

The server speaks JSON-RPC over a Unix domain socket by default. You
can also expose it over HTTP (with bearer-token auth) for remote
clients.

### Clients

The TUI and the desktop app are both thin clients. They render UI,
send RPCs, and subscribe to the server's event stream for streaming
responses. They do not own state — every message, every session, every
provider key lives on the server.

Multiple clients can connect to the same server. You can run the
desktop and one or more TUI windows in parallel and watch updates
land in any of them.

### Tool executor

File read, write, edit, shell, and image rendering run in a separate
sandboxed process from the server. This is the security boundary: the
tool process does not have access to the server's API keys or
configuration. On Linux and WSL, the tool process is wrapped in
Bubblewrap; on macOS, `sandbox-exec` is used. Even within the sandbox,
the tool can only write to directories the agent is explicitly allowed
to write to. See [Sandboxing](sandboxing.md).

### Bridges

Telegram, Discord, and the local voice dictation daemon are optional
add-ons. Each runs as a small process that translates between the
outside world and the server's RPC protocol:

- **Telegram and Discord** translate chat messages to and from
  server RPCs. Each user gets their own session; agent, model, and
  provider can be configured per user.
- **Voice dictation** runs locally on the same machine as the client.
  The client (TUI or desktop) spawns the voice daemon, streams audio
  in, and reads transcribed text out as it lands. See
  [Voice Dictation](voice-dictation.md).

### Plugins

Plugins are extensions that contribute tools, slash commands, or
event-driven behaviour. Some are built into the server (cron,
heartbeat, checklist, background review). Others live in
`~/.nocturnal/plugins/` and can be added or removed without restarting.
See [Plugins](plugins.md).

## Why this shape

**One binary on disk.** The TUI, the server, the tool executor, and the
plugins are all part of a single binary. When you launch `nocturnal`,
the binary dispatches to whichever component the command names
(`nocturnal` → TUI; `nocturnal serve` → server; `nocturnal tool` →
executor; `nocturnal tui` → thin TUI client; and so on). This
simplifies installation and upgrade — there is exactly one file to
download and replace.

**Privilege separation.** The server holds your API keys. The tool
executor never sees them. The sandbox hides the server's
configuration directory from the tool process entirely. An agent that
somehow convinces the LLM to misbehave still cannot read your
credentials or escape the directories you have allowed it to write
to.

**Crash isolation.** A panic in the tool executor does not bring down
the server. A bug in a plugin does not corrupt the session store. The
components are independent processes, and a crash in one only kills
that one — the others keep running.

**Local-first.** Everything runs on your machine. There is no cloud
service in the middle. The server talks directly to whichever LLM
provider you configure. Your code and your conversation history stay
on disk where you control them.

**Unix socket by default.** The server does not open any TCP ports
unless you explicitly configure HTTP mode. A local-only deployment has
no network surface at all — the TUI and the desktop connect over a
Unix domain socket.

## Where state lives

| What | Where |
|---|---|
| Configuration | `~/.nocturnal/config.yaml` |
| API keys | `~/.nocturnal/secrets/.env` (private to the server) |
| Sessions | `~/.nocturnal/sessions/by-uuid/<uuid>/` |
| Built-in agents | `~/.nocturnal/agents.builtin/` (overwritten on upgrade) |
| User agents | `~/.nocturnal/agents/` |
| Project agents | `<project>/.nocturnal/agents/` |
| Plugins | `~/.nocturnal/plugins/` |
| Logs | `~/.nocturnal/logs/` |
| Backups | `~/.nocturnal/backups/latest/` |

Override the base directory with `NOCTURNAL_HOME`. The desktop and the TUI
use the same paths.