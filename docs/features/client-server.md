# Client/Server Architecture

## What

Nocturnal is split into two roles. The **server** runs as `nocturnal serve`
in the background and owns sessions, credentials, provider connections, and
streaming state. The **TUI** runs as plain `nocturnal` (no subcommand) and is
a thin client that connects to the server over a Unix socket and renders
what the server sends.

Multiple TUIs can connect to the same server simultaneously. Each TUI can view
different sessions. You can also run one server per TUI (standalone mode) if
you prefer isolated instances.

## How it works

**Unix socket IPC.** The server listens on a Unix domain socket (no TCP ports
by default). This means no open network ports unless you explicitly configure
HTTP. The socket path defaults to `$NOCTURNAL_HOME/secrets/socket` and can be
overridden with the `NOCTURNAL_LLM_SOCKET` environment variable or the `--socket`
flag.

**The server is the brain.** It manages:

- Session lifecycle (creation, storage, compaction, cleanup)
- LLM provider connections and API keys
- Streaming responses and tool execution
- Subagent spawning and inter-agent messaging
- Plugin system

**The TUI is a renderer.** It sends user input to the server and renders the
responses. It subscribes to sessions and receives streaming updates as they
arrive. Switching sessions is just subscribing to a different stream -- no data
migration or state transfer.

**Shared server.** Multiple TUI processes can connect to the same socket. Each
operates independently -- you can view different sessions in different
terminals. This is useful for monitoring subagents in one window while working
in another.

**Standalone mode.** Pass `--standalone` to the TUI to get a private server
instance with its own abstract namespace socket. The TUI starts the server as a
child process and kills it on exit. This gives you a 1:1 TUI-to-server
relationship with no shared state:

```
nocturnal --standalone
```

When not in standalone mode, the TUI connects to the existing server at the
configured socket path. If no server is running, it starts one automatically.

## Configuration

The server socket is configured via `~/.nocturnal/config.yaml`:

```yaml
server:
  socket: "/path/to/custom/socket"    # Default: $NOCTURNAL_HOME/secrets/socket
  port: 8080                          # Optional HTTP port
  workspace: "prog"                   # Workspace root (relative to $HOME)
```

Environment variables take precedence over config:

- `NOCTURNAL_LLM_SOCKET` -- override the socket path
- `NOCTURNAL_HOME` -- override the base directory (default: `~/.nocturnal`)

### Standalone mode

No configuration needed. Run the TUI with `--standalone`:

```bash
nocturnal --standalone
```

The TUI generates a unique abstract socket name (`@nocturnal-llm-{pid}`) and starts
a private server instance. When the TUI exits, it terminates the server.

### Shared server mode

Start the server explicitly (or let the TUI start it):

```bash
nocturnal serve
```

Then connect from any number of TUIs:

```bash
nocturnal          # connects to default socket
nocturnal --new    # starts a new session
```

All connected TUIs share the same session list and can work on different
sessions concurrently.
