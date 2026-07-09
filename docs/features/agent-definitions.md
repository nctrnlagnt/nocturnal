# Agent Definitions

## What

Agents are configured profiles that determine how a session behaves: which model
to use, which tools are available, what it can write to, and which subagents it
can spawn. They are defined as markdown files with YAML frontmatter. The server
scans three directories in priority order (later overrides earlier by `name`):

| Priority | Directory                  | Managed by       | Purpose                  |
|----------|----------------------------|------------------|--------------------------|
| Lowest   | `~/.nocturnal/agents.builtin/`  | Installer       | Built-in shipped defaults |
| Middle   | `~/.nocturnal/agents/`          | User             | Personal agents          |
| Highest  | `<project>/.nocturnal/agents/`  | User/project     | Project-specific agents  |

To work with this layering, edit files in `~/.nocturnal/agents/` (never in
`agents.builtin/`, which the installer overwrites on upgrade). To override a
built-in agent, copy it from `agents.builtin/` into `agents/` (same `name`)
and edit the copy; to hide a built-in without removing it, prefix the file
name with a leading dot.

## How it works

Each agent is a `.md` file with a YAML frontmatter block followed by the
system prompt body. The frontmatter controls capabilities; the body provides the
agent's instructions.

Example agent file:

```markdown
---
name: code
description: General-purpose coding assistant.
model: code
provider: alias
agents: [review, investigate, edit]
tools: [read, write, edit, bash, sessions_spawn, sessions_await]
writes_restricted: []
network: false
temperature: 0.0
---

You are an expert software engineer...
```

Shipped and global user-defined agents use the convention
`provider: alias, model: <agent-name>` (e.g. agent `code` references
`alias/code`). That alias is the configurable handle for the agent's
model — see [Multi-Provider Aliases](multi-provider-aliases.md) for the
full alias resolution rules.
Project-local agents can use any `provider`/`model` form.

### Frontmatter fields

| Field | Required | Description |
|-------|----------|-------------|
| `name` | Yes | Identifier used to reference the agent |
| `description` | Yes | Short description shown in session picker and agent listings |
| `model` | No | Default model for this agent when spawned as a subagent. `null` uses the session default. For shipped/global agents this is the alias name (e.g. `code` → `alias/code`). |
| `provider` | No | Default provider. `null` uses the session default. Shipped/global agents use `alias`. |
| `tools` | Yes | List of tools this agent can use |
| `agents` | No | List of subagents this agent can spawn |
| `skills` | No | List of skill names to load into the agent's context (filtered by `skills.disabled` in `config.yaml`) |
| `writes_restricted` | No | Paths the agent is allowed to write to. Empty means unrestricted |
| `network` | No | Whether the agent can access the network (default: false) |
| `temperature` | No | Sampling temperature for the model |
| `user_selectable` | No | Whether the agent appears in the TUI session picker (default: true) |

### Agent provider/model as a fallback default

The agent definition's `provider` and `model` are a **fallback default
applied at subagent spawn time**. They are **not** an override for
top-level sessions.

Resolution priority for a session's provider/model:

1. `state.alias_resolved` (the alias the user has selected)
2. `state.provider`/`state.model` (the user's most recent selection)
3. Caller override (e.g. parent session passing values to a child)
4. Parent session's provider/model (for inherited child sessions)
5. `.meta.json` (persisted from the session's last execution)
6. Message history (last provider/model used in this session)
7. **Agent definition's `provider`/`model`** (fallback default — only
   reached when nothing above provides a value)
8. `config.defaults`

This means the agent's `provider: alias, model: code` is what spawns a
fresh subagent using `alias/code`. It does **not** override the user's
selection on a top-level session, even if the alias has no configured
target.

### System prompt body

Everything after the frontmatter `---` becomes the agent's system prompt. You
can include other agent files with `@filename.md` references, which are inlined
at runtime. Built-in agents may include `@common.md` or `@common/foo.md`; for
overrides in `~/.nocturnal/agents/`, these references fall back to `agents.builtin/`
so a copied override continues to work.

### Bundled agents

Nocturnal ships with several built-in agents (in `agents.builtin/`):

**code** — General-purpose coding assistant. Can read, write, edit files, run
commands, and spawn specialized subagents. Spawned by the default `coworker`
agent when it needs to delegate hands-on coding work.

**plan** — Architectural planner. Produces high-level design documents and task
breakdowns. Writes are restricted to the `.plans/` subdirectory — it cannot
modify source code. Other agents can call on the plan agent for higher-quality
architectural output than simple prompt-based planning.

**review** — Uncompromising code reviewer. Read-only access. Analyzes code for
logical flaws, factual errors, omissions, and inefficiencies. Cannot make
changes.

**investigate** — Exploration and diagnostics agent. Read-only access. Traces
issues, examines logs, and reports findings without modifying anything.

**edit** — Lightweight editing agent. Uses a cheaper, faster model for precise
file edits. Designed to be called by other agents to apply changes while keeping
the parent conversation clean. Not user-selectable (only spawned by other
agents).

## Configuration

Agent files live in any of the three directories above. To create a custom
agent:

1. Create a new `.md` file in `~/.nocturnal/agents/` (or `<project>/.nocturnal/agents/`)
2. Add YAML frontmatter with the fields below
3. Write the system prompt body
4. The agent becomes available immediately (use `/reload` if it doesn't appear)

To override a built-in, create a file with the same `name` in a higher-priority
directory. To share common instructions across agents, place them in
`~/.nocturnal/agents/common/` and reference them with `@common/filename.md`.

### Restricting write access

The `writes_restricted` field controls where an agent can create or modify
files. This is useful for agents that should only produce specific output:

```yaml
# Plan agent can only write to .plans/
writes_restricted: [".plans"]

# Read-only agent
writes_restricted: []

# Unrestricted (can write anywhere)
writes_restricted: null
```
