# CLI/Scripting Mode

## What

The `nocturnal` binary can be used non-interactively for scripting and automation.
CLI mode connects to a running server, executes a command, and exits — no TUI required.

## How it works

When any CLI flag is passed, the TUI is bypassed entirely. The binary connects to the
nocturnal server over the Unix socket, sends a request, prints the result to stdout,
and exits. This makes it suitable for shell scripts, cron jobs, and automation pipelines.

CLI mode requires a running server (or will start one automatically). Use `--standalone`
for isolated instances that don't interfere with your main TUI session.

## CLI Flags

### Session Management

| Flag | Description |
|------|-------------|
| `--session-new` | Create a new session. Prints the session ID to stdout. |
| `--session-list` | List all sessions. Output: `ID\tLabel\tStatus\tTokens` |
| `--session-delete <id>` | Delete a session by ID. |
| `--session-send <msg>` | Send a message to a session. Requires `--session-id`. |
| `--session-id <id>` | Target session ID for `--session-send`. |

### Information Queries

| Flag | Description |
|------|-------------|
| `--agents` | List available agents. One per line. |
| `--models` | List available models. Output: `provider/model\tName` |
| `--logs <id\|all>` | Subscribe to session logs (streaming). |

### Session Options

| Flag | Description |
|------|-------------|
| `--agent <name>` | Agent to use for new session. |
| `--model <name>` | Model to use. |
| `--system-prompt <text>` | System prompt for new session. |
| `--no-reply` | With `--session-send`, exit without waiting for response. |
| `--if-idle` | With `--session-send`, only send if session is not active. |
| `--debug-prompt` | Print the complete system prompt to stderr. |
| `--show-reasoning` | Show reasoning content in output. |

### Connection Flags

| Flag | Description |
|------|-------------|
| `--socket <path>` | Path to Unix socket. Default: `$NOCTURNAL_HOME/secrets/socket` |
| `--http <addr>` | HTTP address for nocturnal server. |
| `--standalone` | Run with a private server instance (unique abstract socket). |

### TUI Flags (also work in CLI mode)

| Flag | Description |
|------|-------------|
| `--resume <id>` | Resume a specific session by ID. |
| `--no-welcome` | Skip welcome image on startup. |
| `--debug` | Enable debug logging to stderr. |
| `--perf-marker <path>` | Write timestamped markers to file for performance testing. |
| `-h, --help` | Print help text. |

## Environment Variables

| Variable | Description |
|----------|-------------|
| `NOCTURNAL_LLM_SOCKET` | Override socket path (lower priority than `--socket`). |
| `NOCTURNAL_HOME` | Override base directory. Default: `~/.nocturnal` |
| `NOCTURNAL_AUTH_TOKEN` | Authentication token for HTTP connections. |
| `NOCTURNAL_LLM_BINARY` | Path to nocturnal binary (for auto-starting server). |
| `EDITOR` | Editor for opening files (used in TUI mode). |

## Examples

### Create a new session

```bash
$ nocturnal --session-new
abc123-def456-ghi789
```

### List sessions

```bash
$ nocturnal --session-list
abc123-def456-ghi789	My Project	idle	1234 tok
xyz789-abc123-def456	Another Session	streaming	5678 tok
```

### Send a message to a session

```bash
$ nocturnal --session-send "What is 2+2?" --session-id abc123-def456-ghi789
Message sent to abc123-def456-ghi789
```

### List available agents

```bash
$ nocturnal --agents
default
code-assistant
reviewer
```

### List available models

```bash
$ nocturnal --models
openai/gpt-4o	GPT-4o
openai/gpt-4o-mini	GPT-4o Mini
anthropic/claude-sonnet-4-20250514	Claude Sonnet 4
```

### Delete a session

```bash
$ nocturnal --session-delete abc123-def456-ghi789
Session abc123-def456-ghi789 deleted
```

### Use standalone mode for isolated automation

```bash
$ nocturnal --standalone --session-new
xyz789-abc123-def456
```

Standalone mode starts a private server with a unique abstract socket
(`@nocturnal-llm-{pid}`). The server is terminated when the command completes.

## Scripting Workflows

### Create session and send initial prompt

```bash
#!/bin/bash
SESSION_ID=$(nocturnal --session-new)
nocturnal --session-send "Analyze the codebase" --session-id "$SESSION_ID"
echo "Started session: $SESSION_ID"
```

### Check if a session is idle before sending

```bash
#!/bin/bash
SESSION_ID="abc123-def456-ghi789"
nocturnal --session-send "Continue" --session-id "$SESSION_ID" --if-idle
```

### Use a specific agent and model

```bash
$ nocturnal --session-new --agent code-assistant --model anthropic/claude-sonnet-4-20250514
```

### Custom system prompt

```bash
$ nocturnal --session-new --system-prompt "You are a code reviewer. Be concise."
```

### Debug the system prompt

```bash
$ nocturnal --session-send "Hello" --session-id abc123 --debug-prompt 2>&1 | head -100
```

### Parse session list in scripts

```bash
#!/bin/bash
# Find idle sessions
nocturnal --session-list | awk '$3 == "idle" { print $1 }'
```

## Integration with Cron

```cron
# Send a daily summary request at 9 AM
0 9 * * * nocturnal --session-send "Generate daily summary" --session-id abc123 --no-reply
```

## Notes

- CLI mode connects to an existing server by default. Use `--standalone` for isolated instances.
- The `--logs` flag streams output until interrupted (Ctrl+C).
- Session IDs are UUIDs. Use the full ID or a unique prefix.
- Output formats are tab-separated for easy parsing with `awk` or `cut`.