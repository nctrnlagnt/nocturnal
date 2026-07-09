# Subagents

## What

Nocturnal supports a multi-agent system where one agent can spawn other agents
to work on tasks in parallel. Subagents can be ephemeral (short-lived, for a
single task) or persistent (long-lived, with their own session). The TUI's
subagent panel gives you real-time visibility into what every agent is doing.

## How it works

**Ephemeral subagents.** The default mode. An agent spawns a subagent for a
specific task, and the subagent automatically sends its result back to the
parent when complete. The parent does not need to poll -- results are delivered
automatically. Ephemeral subagents are read-only: they cannot spawn their own
subagents or send arbitrary messages.

**Persistent agents.** Long-lived agents with their own sessions. They can be
messaged by other agents, spawn their own subagents, and maintain state across
multiple interactions. Use persistent agents for ongoing tasks that need to
accept incoming messages over time.

**Non-blocking execution.** When a parent spawns a subagent, the parent
continues working. It does not wait for the child to finish. This lets agents
delegate multiple tasks in parallel and collect results as they arrive.

**Inter-agent messaging.** Agents can send messages to other sessions using the
`send_message` tool. Messages are delivered on the recipient's next turn -- no
polling or sleeping required. Use `agent:parent` to message the parent session,
or `agent:<session-id>` to message a specific persistent agent.

**Agent swarms, teams, and fans.** Agents can spawn multiple subagents in
sequence to parallelize work -- for example, researching several topics
simultaneously, or running independent code investigations across different
parts of a project. The parent coordinates by sending tasks and collecting
results.

**Session forking.** A subagent can optionally inherit the parent's full
conversation context (`share_context: true`). This creates a fork of the
parent's session, letting the child explore a different direction without
affecting the parent. Useful for parallel investigations of the same codebase.

**Depth limits.** There is a maximum spawn depth to prevent runaway agent
creation. At the depth limit, `sessions_spawn` is hidden from the agent's tool
list.

## Configuration

Agents are defined as markdown files with YAML frontmatter. Place them in
`~/.nocturnal/agents/` (global) or `<project>/.nocturnal/agents/` (project-specific).
Project agents override home agents by name.

### Agent file format

```markdown
---
name: coder
description: General-purpose coding agent
model: claude-sonnet-4-20250514
tools: [read, write, edit, bash]
writes_restricted: [src, tests]
network: false
agents: [researcher, reviewer]    # Which agents this one can spawn
---

You are a coding assistant. Follow the project's coding conventions.
```

Key frontmatter fields:

| Field | Description |
|-------|-------------|
| `name` | Agent identifier (required) |
| `description` | Short description shown in the agent picker |
| `model` | Override the default model for this agent |
| `tools` | Which tools the agent can use. `None` = all, `[]` = none |
| `writes_restricted` | Restrict filesystem writes. `None` = unrestricted, `[]` = read-only |
| `network` | Whether the agent has network access (default: true) |
| `agents` | Which agents this one may spawn. `None` = all, `[]` = none |
| `user_selectable` | Show in the TUI agent picker (default: true) |
| `provider_side_tools` | Provider-specific tools to pass through (e.g., `web_search_preview`) |

### Internal agents

Agents placed in the `_internal/` subdirectory are not shown in the TUI agent
picker but can still be spawned by other agents. Use these for background tasks
that the user shouldn't select directly.

## Usage

### Subagent panel

When a session has active subagents, the TUI shows a panel below the viewport.
In collapsed mode it shows a one-line summary:

```
subagents: 3 active, 47 completed (ctrl+x a to expand)
```

Expand it to see details:

```
┌─ shift+↓ to view ──────────────────────────────────┐
│ ◎ → research    1.2k tok  web_search  "found 3..." │
│ ●   coder       500 tok   edit        "fixed bug"  │
│ ✓   reviewer    800 tok   read        "approved"   │
└────────────────────────────────────────────────────┘
```

Each row shows:

- **Status glyph** -- active, completed, error, rate-limited, etc.
- **Label** -- the agent's name or custom label
- **Token count** -- context size so far
- **Last tool call** -- most recent tool used
- **Last output** -- truncated last line of output

### Keyboard navigation

| Key | Action |
|-----|--------|
| Shift+Arrow Down | Open subagent panel / enter navigation mode |
| Shift+Arrow Left/Right | Cycle through subagents |
| Shift+Arrow Up | Return to parent session |
| Enter (on selected agent) | View that agent's session |

While viewing a subagent session, the viewport shows its full conversation.
Press Shift+Arrow Up to return to the parent.

### Spawning subagents (from an agent)

Agents use the `sessions_spawn` tool:

```
sessions_spawn(
  agent: "researcher",
  task: "Find all uses of deprecated API functions",
  mode: "ephemeral",        # or "persistent"
  label: "api-deprecation", # optional display label
  share_context: true       # optional: inherit parent's history
)
```

### Sending messages between agents

```
send_message(
  to: "agent:parent",       # or "agent:<session-id>"
  content: "Research complete. Found 12 deprecated calls."
)
```

## Advanced Options

The following options are available for specialized use cases. Most
agents will not need these features.

### Time limits

Set a timeout for spawned sessions using `time_limit_secs`. When the
limit is reached, the session is interrupted automatically.

```
sessions_spawn(
  agent: "researcher",
  task: "Search for API documentation",
  label: "api-search",
  time_limit_secs: 300  # 5 minute timeout
)
```

This is useful for bounding long-running tasks and preventing runaway
agent execution. The timeout applies to the entire session lifetime,
not individual tool calls.

### Inactive spawns

Create a session without activating it using `inactive: true`. The
session is created in `Waiting` status and activates only when it
receives a message.

```
sessions_spawn(
  agent: "worker",
  task: "Process queued items",
  label: "queue-worker",
  inactive: true
)
```

Use this for:

- Pre-creating sessions that will be activated later
- Setting up persistent agents before sending work
- Deferring execution until a trigger message arrives

**Note:** Inactive spawns cannot use `system_task` or `share_context`.
These features require in-memory message queuing, which conflicts with
inactive spawn's disk-persistence model.

### System reminders

Inject a `<SYSTEM>`-tagged reminder into the spawned session using
`system_task`. This message appears as a system reminder (like the
operational directives) and is useful for injecting constraints or
context that the agent should treat as authoritative.

```
sessions_spawn(
  agent: "coder",
  task: "Implement the feature",
  label: "feature-impl",
  system_task: "You must not modify files outside src/. All changes must pass tests."
)
```

The system reminder is delivered before the task message, ensuring the
agent sees the constraints before starting work.

### Delivery target override

Change where ephemeral results are delivered using `respond_to`. By
default, ephemeral agents send their result to the parent session.
This option redirects delivery to a different session.

```
sessions_spawn(
  agent: "researcher",
  task: "Gather data",
  label: "data-gather",
  respond_to: "agent:coordinator-session-id"
)
```

Use this for:

- Routing results through a coordinator session
- Fan-in patterns where multiple agents report to one aggregator
- Complex multi-agent workflows with custom routing

### Concurrency limits

The server can limit how many concurrent child sessions a parent may
have. Configure this in `~/.nocturnal/config.yaml`:

```yaml
server:
  max_session_concurrency: 4
```

When the limit is reached, `sessions_spawn` returns an error:

```json
{
  "error": "concurrency_limit",
  "message": "Session concurrency limit reached (4/4). Wait for a child session to complete before spawning more."
}
```

This helps reserve provider concurrency for other sessions and prevents
one parent from monopolizing resources.

### Depth limits

The server enforces a maximum spawn depth to prevent runaway agent
creation. Configure this in `~/.nocturnal/config.yaml`:

```yaml
server:
  max_spawn_depth: 3
```

At the depth limit, `sessions_spawn` is hidden from the agent's tool
list. The agent cannot spawn children at all. This prevents infinite
recursion and bounds the agent tree depth.

### Write restriction inheritance

Child sessions inherit the intersection of the parent's and agent's
write restrictions. This ensures children cannot write outside the
parent's allowed paths, even if the agent definition specifies broader
access.

Example:

- Parent restricted to `["src", "docs"]`
- Agent definition restricted to `["src", "tests", "examples"]`
- Child effective restriction: `["src"]` (intersection)

Special cases:

- Parent read-only (`[]`) → child is read-only regardless of agent
- Agent read-only (`[]`) → child is read-only regardless of parent
- Parent unrestricted (`None`) → child uses agent's restriction
- Agent unrestricted (`None`) → child uses parent's restriction

This security model ensures that spawned agents cannot escape the
parent's sandbox, maintaining the privilege boundary established by
the spawning session.