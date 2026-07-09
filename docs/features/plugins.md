# Plugins

## What

Plugins extend nocturnal with tools, slash commands, and event-driven behavior. They run
in isolation from the core server, communicating via a JSON wire protocol. Plugins can
provide new capabilities to agents, respond to session events, and integrate external
services.

## How it works

Plugins are discovered at startup from `~/.nocturnal/plugins/`. Each plugin is a directory
containing a `plugin.yaml` manifest and the executable code (script, binary, etc.).

There are two types of plugins:

- **Built-in plugins** — Compiled into nocturnal (cron, heartbeat, checklist, background-review).
  These are always available and run in-process.
- **External plugins** — Subprocesses that communicate via stdin/stdout. These can be
  written in any language and are discovered from `~/.nocturnal/plugins/`.

Plugins can:

- **Provide tools** — Functions that agents can call during conversations
- **Handle slash commands** — Custom `/commands` typed in the TUI
- **Subscribe to events** — React to session lifecycle (status changes, tool calls, etc.)
- **Respond to hooks** — Inject content into system prompts or compaction summaries

## Plugin discovery

Plugins are discovered from:

1. **Built-in plugins** — Always loaded at startup (unless disabled)
2. **`~/.nocturnal/plugins/<name>/`** — External plugin directories containing a `plugin.yaml`

On startup, nocturnal scans `~/.nocturnal/plugins/` for directories containing a `plugin.yaml`
file. Each valid manifest is parsed and the plugin is spawned (if `enabled: true`).

## Plugin manifest (plugin.yaml)

The manifest declares the plugin's command, tools, events, and visibility rules:

```yaml
name: my-plugin
command: ["python3", "-u", "main.py"]
hidden_when: never          # never | ephemeral | always
enabled: true               # set to false to disable
event_subscriptions:
  - status
  - response_complete
slash_commands:
  - name: mycmd
    description: "Do something"
    usage: "[arg]"
hook_subscriptions:
  - compaction_addendum
```

### Fields

| Field | Required | Description |
|-------|----------|-------------|
| `name` | Yes | Human-readable plugin name |
| `command` | Yes (external) | Command to launch the plugin. Can be a string or array |
| `hidden_when` | No | Tool visibility: `never` (always visible), `ephemeral` (hidden for ephemeral sessions), `always` (always hidden, daemon-only) |
| `enabled` | No | Whether to spawn the plugin. Defaults to `true` |
| `event_subscriptions` | No | Event types to receive (e.g., `status`, `chunk`, `tool_call`) |
| `slash_commands` | No | Slash commands this plugin handles |
| `hook_subscriptions` | No | Hook methods to respond to (e.g., `compaction_addendum`) |

### Command formats

Single string (shell resolution via PATH):

```yaml
command: "my-plugin-binary"
```

Array (explicit arguments, no shell):

```yaml
command: ["python3", "-u", "main.py"]
```

The working directory is set to the plugin directory.

## Installing plugins

To install an external plugin:

1. Create a directory in `~/.nocturnal/plugins/`:

   ```bash
   mkdir -p ~/.nocturnal/plugins/my-plugin
   ```

2. Add your plugin code (scripts, binaries, etc.)

3. Create a `plugin.yaml` manifest:

   ```yaml
   name: my-plugin
   command: ["./my-plugin"]
   hidden_when: never
   ```

4. Reload plugins or restart the server:

   ```
   /plugins reload
   ```

### Example: Python plugin

Directory structure:

```
~/.nocturnal/plugins/hello/
├── plugin.yaml
└── hello.py
```

`plugin.yaml`:

```yaml
name: hello
command: ["python3", "-u", "hello.py"]
hidden_when: ephemeral
```

`hello.py` sends a handshake on stdout, then handles incoming JSON messages.

## The /plugins command

Manage plugins from the TUI:

| Command | Description |
|---------|-------------|
| `/plugins list` | Show all plugins with their status |
| `/plugins <name> enable` | Enable and start a plugin |
| `/plugins <name> disable` | Stop and disable a plugin |
| `/plugins <name> status` | Detailed info for one plugin |
| `/plugins reload` | Rescan for new plugins |

### Plugin states

| State | Description |
|-------|-------------|
| `builtin` | Built-in plugin, running in-process |
| `running` | External plugin, subprocess active |
| `disabled` | Explicitly disabled via `enabled: false` |
| `stopped` | Process exited cleanly |
| `crashed` | Process exited unexpectedly |
| `starting` | Transient state during spawn |

Disabling a plugin persists `enabled: false` to its `plugin.yaml`. The plugin
won't be spawned on next server start.

## Built-in plugins

These plugins are compiled into nocturnal and always available:

### cron

Scheduled and recurring tasks. Agents can create jobs that run on a schedule
or after a delay. Each job spawns a fresh ephemeral session.

See [Cron](cron.md) for details.

### heartbeat

Periodic agent check-ins. Sends a nudge to idle sessions at regular intervals,
keeping the agent engaged in long-running tasks.

See [Heartbeat](heartbeat.md) for details.

### checklist

Structured task execution with mandatory review. Agents create a plan and
checklist, work through items, then submit for review before completion.

See [Checklist Mode](checklist-mode.md) for details.

### background-review

Automatic memory extraction. Spawns a review agent after N turns to extract
valuable information into USER.md and MEMORY.md files.

See [Cross-Session Memory](cross-session-memory.md) for details.

## Advanced: Plugin hooks

Plugins can subscribe to hooks that inject content into the agent's context:

| Hook | When called | What it returns |
|------|-------------|-----------------|
| `compaction_addendum` | During context compaction | Text to append to the summary |
| `system_prompt_addendum` | When building system prompt | Text to append to the prompt |

Hooks are called with a timeout. If the plugin doesn't respond in time, the
hook is skipped. Only subscribe to hooks if your plugin needs to inject
dynamic content.

## Advanced: Event subscriptions

Plugins can subscribe to session events:

| Event | When fired |
|-------|------------|
| `status` | Session status changes |
| `user_message` | User message saved to session |
| `tool_call` | Tool invocation |
| `tool_result` | Tool result received |
| `chunk` | Streaming content chunk |
| `response_complete` | Turn finished |
| `session_compacted` | Context compaction complete |
| `external_message` | Message from external platform |

For the full event reference, see the plugin SDK documentation in the Nocturnal Plugin SDK release.

## Advanced: Messaging daemons

Plugins like `nocturnal-telegramd` and `nocturnal-discordd` run as background daemons that
bridge external platforms to nocturnal sessions. They use:

- `hidden_when: always` — No tools exposed to agents
- Event subscriptions for `external_message`, `chunk`, `response_complete`

Incoming messages flow: Platform → plugin → `CreateSession` + `QueueMessage`.

Outgoing messages flow: nocturnal events → plugin → Platform API.

### Handshake fields

After spawning, external plugins send a handshake JSON on stdout to declare
runtime configuration. These fields are **not** in the manifest:

| Field | Description |
|-------|-------------|
| `tools` | Tool definitions the plugin provides (sent at runtime, not in manifest) |
| `event_prefix_filter` | Filter events by prefix for messaging daemons (e.g., `"telegram:"`) |

The handshake is sent by the plugin SDK automatically. For the full handshake reference, see the Nocturnal Plugin SDK release.