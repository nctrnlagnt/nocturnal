# Configuration Reference

## Overview

Nocturnal uses a YAML configuration file at `~/.nocturnal/config.yaml` for server settings, providers, aliases, and tool execution. API keys and secrets are stored separately in `~/.nocturnal/secrets/.env` for security.

## Configuration File Location

The main config file is located at:

| Path | Description |
|------|-------------|
| `~/.nocturnal/config.yaml` | Main server configuration |
| `~/.nocturnal/secrets/.env` | API keys and secrets (private) |
| `~/.nocturnal/tui.conf` | TUI preferences (JSON, managed by TUI) |

The base directory can be overridden with the `NOCTURNAL_HOME` environment variable.

---

## Server Configuration

The `server` section controls core server behavior.

```yaml
server:
  socket: /run/nocturnal-llm/socket
  port: 8080
  worker_threads: 8
  max_spawn_depth: 3
  max_session_concurrency: 5
  workspace: /home/user/projects
  suppress_include_markers: true
```

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `socket` | string | `$NOCTURNAL_HOME/secrets/socket` | Unix socket path for TUI connection. Override with `NOCTURNAL_LLM_SOCKET` env var. |
| `port` | integer | `8080` | HTTP port for API access (optional, for external clients) |
| `worker_threads` | integer | `8` | Number of internal worker threads for the server. The server is I/O-bound, so 8 suffices even on high-core machines. |
| `max_spawn_depth` | integer | none (unlimited) | Maximum spawn depth for ephemeral agents. Agents at or beyond this depth cannot use `sessions_spawn`. |
| `max_session_concurrency` | integer | none (unlimited) | Maximum concurrent child sessions per parent. Limits sub-agent parallelism to reserve provider concurrency. |
| `workspace` | string | unset | Default working directory on the server for sessions created without explicit cwd. Accepts absolute, `~/...`, or home-relative paths. Set during first-run onboarding (TUI/Dialog or Desktop/Settings). Required for plugin-created sessions (Telegram, Discord, cron) that don't supply a path. Stored as home-relative when under `$HOME` (e.g. `prog`), else absolute. |
| `suppress_include_markers` | bool | `true` | Suppress `[Included file: ...]` markers in `@include` processed content. |

---

## Provider Configuration

Providers are defined under the `providers` key. Each provider connects to an LLM API.

```yaml
providers:
  anthropic:
    type: anthropic-compatible
    base_url: https://api.anthropic.com
    api_key_env_var: ANTHROPIC_API_KEY
    default_model: claude-sonnet-4-20250514
    models:
      - id: claude-sonnet-4-20250514
        name: Claude Sonnet 4
        input_price: 3.0
        output_price: 15.0
        context_limit: 200000
        output_limit: 16384
        effort: high

  openai:
    type: openai-compatible
    base_url: https://api.openai.com/v1
    api_key_env_var: OPENAI_API_KEY
    default_model: gpt-4o
```

### Provider Types

| Type | Protocol | Use Case |
|------|----------|----------|
| `openai-compatible` | OpenAI Chat Completions API | OpenAI, most third-party LLMs |
| `openai-responses-compatible` | OpenAI Responses API | ChatGPT OAuth (codex) |
| `anthropic-compatible` | Anthropic Messages API | Anthropic Claude, third-party Claude-compatible gateways |
| `google` | Google Generative AI API | Gemini models |
| `test` | Built-in test provider | Development/testing (no API key needed) |

### Provider Fields

| Field | Required | Description |
|-------|----------|-------------|
| `type` | Yes | Provider type from table above |
| `base_url` | Yes* | API endpoint URL. Defaults provided for known providers. |
| `api_key_env_var` | Yes* | Name of the environment variable that holds this provider's API key. The actual key value lives in `~/.nocturnal/secrets/.env` (or the process environment). `config.yaml` stores the variable name only — it never holds a literal key. |
| `default_model` | No | Model used when no model specified for this provider |
| `api_path` | No | Custom API path suffix (e.g., `/v1/chat/completions`) |
| `models` | No | List of available models (required for Anthropic-compatible without model list endpoint) |

*Not required for `test` provider type.

### Model Configuration

Each model in the `models` list can have these fields:

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Model identifier (required) |
| `name` | string | Display name |
| `description` | string | Model description |
| `input_price` | number | Cost per million input tokens (USD) |
| `output_price` | number | Cost per million output tokens (USD) |
| `context_limit` | integer | Maximum context window size |
| `output_limit` | integer | Maximum output tokens |
| `effort` | string | Anthropic effort level: `low`, `medium`, `high`, `xhigh`, `max` |

---

## Alias Configuration

Aliases map short names to provider/model combinations. Every alias uses the
same object shape with a `targets` list and optional selection strategy. Targets
can be provider/model strings, effort objects, or a single `alias/<name>`
reference to another alias.

```yaml
aliases:
  # Single target
  fast:
    targets: [anthropic/claude-haiku-4-20250514]

  # Multi-provider (quota failover)
  smart:
    targets:
      - anthropic/claude-sonnet-4-20250514
      - openai/gpt-4o

  # Purpose mapping to another alias
  edit:
    targets: [alias/smart]
```

### Single-Provider Aliases

A single target in the normalized `targets` list:

```yaml
aliases:
  fast:
    targets: [anthropic/claude-haiku-4-20250514]
```

Use as `alias/fast` in model selection.

### Multi-Provider Aliases

A list of targets for quota failover:

```yaml
aliases:
  smart:
    targets:
      - anthropic/claude-sonnet-4-20250514
      - openai/gpt-4o
```

When one target hits a quota limit, the session automatically switches to the next target. See [Multi-Provider Aliases](multi-provider-aliases.md) for details.

---

## Skills Configuration

Skills are auto-discovered from `~/.nocturnal/skills/`, `~/.agents/skills/`, and
the active project's `.agents/skills/`. The `skills` section lets the user
disable individual skill names without removing them from disk.

```yaml
skills:
  disabled:
    - rust-skills
    - zig
```

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `disabled` | string list | `[]` | Skill names that should not be loaded into the agent's system prompt. |

A disabled skill is filtered out at load time — it does not appear in the
system prompt and is not invokable from agent definitions that reference it
by name. Disabling does not delete the skill from disk; re-enabling is a
config edit and a reload away.

The Settings UI exposes a per-skill enabled toggle (one card per discovered
skill) that writes to this same `skills.disabled` list. The schema mirrors
the plugins tab: each skill is a child of `skills/<name>` in the config tree,
and toggling writes `{enabled: false}` (or removes the entry).

See [Agent Skills](agent-skills.md) for the skill authoring and discovery
guide.

---

## Project Configuration

Project-level settings control write access restrictions.

```yaml
project:
  allowed_write_dirs:
    - .
  # system_write_dirs:           # optional; omit to use product defaults
  #   - ~/.cargo
  #   - target
```

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `allowed_write_dirs` | string list | `["."]` | Directories agents may write to (relative to project root). Unset uses product default `["."]`. `[]` = read-only. |
| `system_write_dirs` | string list | product defaults | Infrastructure/build paths always merged into effective write restrictions at runtime (caches, tool dirs, `target`, etc.). Unset uses product defaults; setting the list fully replaces them. `[]` means no system paths. |

### Semantics

- `allowed_write_dirs` unset: product default `["."]` (agents restricted to project directory; system paths still merged)
- `allowed_write_dirs` `[]` (empty list): read-only by default (system paths still merged)
- `allowed_write_dirs` `["src", "tests"]`: only listed directories writable (plus system paths)
- `system_write_dirs` unset: product defaults (cargo/npm/go caches, `target`, `node_modules`, `build`, tool state dirs, …)
- `system_write_dirs` set: full replacement of the default list
- `system_write_dirs` are never persisted to session `.meta.json`; they are re-merged at runtime

---

## Sandbox Configuration

Controls filesystem and network isolation for tool execution.

```yaml
sandbox:
  enabled: true
  hide_nocturnal_home: true
  allow_gpu: false
```

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `enabled` | bool | `true` | Enable sandboxing. When `false`, tools run without restrictions. |
| `hide_nocturnal_home` | bool | `true` | Block `~/.nocturnal/secrets/` from sandboxed processes. Prevents agents from reading API keys. |
| `allow_gpu` | bool | `false` | Expose GPU device nodes to the sandbox so tools like ffmpeg can use hardware acceleration (QSV/AMF/NVENC on Linux, VideoToolbox on macOS). |

See [Sandboxing](sandboxing.md) for platform-specific details.

---

## Tool Executor Configuration

Controls tool execution behavior.

```yaml
tool_executor:
  container: none
  timeout: 300
  nocturnal_image_binary: /usr/local/bin/nocturnal-image
  image_max_dim: 2000
  thumb_max_dim: 512
  read_line_cache: true
```

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `container` | string | `none` | Container runtime (`none`, `docker`, etc.) |
| `container_args` | string list | `[]` | Additional container runtime arguments |
| `binary_path` | string | none | Path to `nocturnal-tool` binary. Auto-detected if not set. |
| `timeout` | integer | `300` | Tool execution timeout in seconds (5 minutes) |
| `nocturnal_image_binary` | string | auto | Path to `nocturnal-image` for image processing. Auto-detected from PATH or relative path. |
| `image_max_dim` | integer | `2000` | Maximum dimension for base64 JPEG sent to LLM |
| `thumb_max_dim` | integer | `512` | Maximum dimension for thumbnail JPEG |
| `read_line_cache` | bool | `true` | Maintain file content cache for edit validation |

---

## Defaults Configuration

Sets the default provider and model for new sessions.

```yaml
defaults:
  provider: anthropic
  model: claude-sonnet-4-20250514
```

| Setting | Type | Description |
|---------|------|-------------|
| `provider` | string | Default provider name |
| `model` | string | Default model ID |

---

## Environment Variables

Environment variables control paths and behavior. They can be set in your shell profile or passed to the process.

| Variable | Default | Description |
|----------|---------|-------------|
| `NOCTURNAL_HOME` | `$HOME/.nocturnal` | Base directory for all Nocturnal state files |
| `NOCTURNAL_LLM_SOCKET` | `$NOCTURNAL_HOME/secrets/socket` | Unix socket path for TUI-server communication |
| `NOCTURNAL_CACHE_DIR` | `$NOCTURNAL_HOME/cache` | Cache directory for model info, provider data |
| `HOME` | (system) | Fallback for `NOCTURNAL_HOME` if not set |
| `EDITOR` | (system) | Editor for `Ctrl+X e` and clicking file paths in TUI |

### Resolution Order

For `api_key_env_var` resolution in `config.yaml`:

1. `~/.nocturnal/secrets/.env` value (highest priority)
2. Process environment (shell profile)

This allows `.env` to override shell values without modifying your `.bashrc`.

---

## TUI Configuration

TUI preferences are stored in `~/.nocturnal/tui.conf` (JSON format). Managed through the TUI itself—no manual editing required.

```json
{
  "theme": "dark",
  "show_thinking": true,
  "show_details": false,
  "scroll_speed": 3,
  "favorites": ["anthropic/claude-sonnet-4", "openai/gpt-4o"],
  "welcome_mode": "animated",
  "movie_loop": true,
  "last_agent": "anthropic/claude-sonnet-4",
  "last_model": "anthropic/claude-sonnet-4-20250514"
}
```

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `theme` | string | `dark` | Color theme name |
| `system_colors` | string | (empty) | Detected terminal colors for System theme: `"rrggbb,rrggbb"` (fg,bg) |
| `system_palette` | string | (empty) | Detected ANSI palette: 16 comma-separated hex triples |
| `show_thinking` | bool | `true` | Display thinking/reasoning blocks |
| `show_details` | bool | `false` | Display detail blocks |
| `scroll_speed` | integer | `3` | Lines per scroll event |
| `favorites` | string list | `[]` | Favorite models in `"provider/id"` format |
| `welcome_mode` | string | `animated` | Welcome screen: `"animated"`, `"static"`, `"none"` |
| `movie_loop` | bool | `true` | Loop welcome animation |
| `last_agent` | string | `coworker` | Last selected agent to restore on startup |
| `last_model` | string | (empty) | Last selected model to restore on startup |

### Available Themes

| Theme | Description |
|-------|-------------|
| `dark` | Default dark theme |
| `monokai` | Warm, high-contrast dark |
| `dracula` | Purple-toned dark |
| `nord` | Cool blue-gray dark |
| `tokyonight-storm dark` | Deep blue dark |
| `catppuccin` | Catppuccin dark |
| `catppuccin-macchiato` | Darker Catppuccin variant |
| `catppuccin-frappe` | Medium-dark Catppuccin |
| `system` | Uses terminal's color palette |
| `clawed` | High-contrast theme |

Switch themes with `Ctrl+P` → "theme" or `Ctrl+X` → `t`.

---

## Dot-Env File

API keys and secrets are stored in `~/.nocturnal/secrets/.env`. This file is private—variables are not injected into the process environment.

```
# API keys
ANTHROPIC_API_KEY=sk-ant-...
OPENAI_API_KEY=sk-...
GOOGLE_API_KEY=AIza...

# Optional export prefix (stripped)
export ANTHROPIC_API_KEY=...
```

### Format

- Lines starting with `#` are comments
- Blank lines ignored
- `KEY=VALUE` sets a variable
- Values may be quoted (single or double)
- `export` prefix is stripped
- Last value wins for duplicate keys

### Security

- Values never injected into `std::env`
- Child processes cannot access via `env`, `printenv`, or `/proc/$PID/environ`
- Sandbox `hide_nocturnal_home` blocks direct file access
- Recommended: `chmod 600 ~/.nocturnal/secrets/.env`

See [Dot-Env File](dot-env.md) for details.

---

## Tool Result Truncation

Tool outputs exceeding size limits are truncated before entering LLM context.

| Limit | Value | Description |
|-------|-------|-------------|
| `MAX_LINES` | 2000 | Maximum lines before truncation |
| `MAX_BYTES` | 50KB | Maximum bytes before truncation |

Truncated output is saved to a temporary file (under the system temp directory) and the response includes a hint pointing to the saved path. Not configurable — hardcoded limits ensure consistent behavior across providers and providers' tool-result formats.

---

## Complete Example

```yaml
# ~/.nocturnal/config.yaml

server:
  worker_threads: 8
  max_spawn_depth: 3
  max_session_concurrency: 5
  workspace: /home/user/projects
  suppress_include_markers: true

providers:
  anthropic:
    type: anthropic-compatible
    base_url: https://api.anthropic.com
    api_key_env_var: ANTHROPIC_API_KEY
    default_model: claude-sonnet-4-20250514
    models:
      - id: claude-sonnet-4-20250514
        name: Claude Sonnet 4
        effort: high
      - id: claude-opus-4-20250514
        name: Claude Opus 4
        effort: max

  openai:
    type: openai-compatible
    base_url: https://api.openai.com/v1
    api_key_env_var: OPENAI_API_KEY
    default_model: gpt-4o

  google:
    type: google
    base_url: https://generativelanguage.googleapis.com
    api_key_env_var: GOOGLE_API_KEY
    models:
      - id: gemini-2.5-pro
        name: Gemini 2.5 Pro

aliases:
  fast:
    targets: [anthropic/claude-haiku-4-20250514]
  smart:
    targets:
      - anthropic/claude-sonnet-4-20250514
      - openai/gpt-4o

defaults:
  provider: anthropic
  model: claude-sonnet-4-20250514

project:
  allowed_write_dirs:
    - .
  # system_write_dirs omitted → product defaults (caches, build dirs, tool state)

sandbox:
  enabled: true
  hide_nocturnal_home: true
  allow_gpu: false

tool_executor:
  timeout: 300
  image_max_dim: 2000
  read_line_cache: true
```

---

## Hot Reload

Use `/reload` in the TUI to re-read `config.yaml` and `.env` without restarting:

1. Edit configuration files
2. Type `/reload`
3. New requests use updated settings
4. In-flight requests continue on old instances

Provider instances are rebuilt, aliases updated, and `.env` re-parsed.
