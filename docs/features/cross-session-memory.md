# Cross-Session Memory

## What

Nocturnal remembers things about you and your projects across sessions. When you
start a new conversation, the system prompt includes two persistent files:

- **User Profile** — your preferences, style, pet peeves (shared across all
  projects in a workspace)
- **Project Memory** — conventions, patterns, lessons learned (per-project)

These files are maintained automatically by a background review process that
watches your conversations and extracts useful information. You can also edit
them directly.

## Where the files live

| File | Location | Scope |
|------|----------|-------|
| `USER.md` | `<workspace>/.nocturnal/USER.md` | Workspace — applies to all projects under the workspace |
| `MEMORY.md` | `<project>/.nocturnal/memory/MEMORY.md` | Per-project |

The **workspace** is the server's configured working directory (the `workspace`
field in server config, or the server's CWD). It's typically a parent directory
containing multiple projects:

```
~/prog/                        # workspace root
~/prog/.nocturnal/USER.md           # global user profile
~/prog/thing1/.nocturnal/memory/MEMORY.md  # project-specific memory
~/prog/thing2/.nocturnal/memory/MEMORY.md  # another project's memory
```

For backward compatibility, if `<workspace>/.nocturnal/USER.md` doesn't exist, the
system falls back to `~/.nocturnal/USER.md`.

## Format

Both files use markdown bullet points — one fact per line:

```markdown
- Prefers brief responses without filler
- Project uses Rust with tokio for async
- Always uses `cargo clippy` before committing
```

There are soft character limits: 1500 for USER.md, 3000 for MEMORY.md. When
the limit is reached, the least valuable entries are removed to make room.

## How it works

**Sessions receive memory at startup.** When a primary session starts (or when
the conversation is compacted), the system reads the memory files and includes
them in the system prompt. The agent sees your preferences and project
knowledge from the very first message.

**Memory is only refreshed when the cache is cold.** LLM providers cache the
system prompt to reduce latency and cost. Changing the prompt mid-conversation
would invalidate that cache. So memory is only re-read when a cache miss is
already guaranteed — at session start, after compaction, or when switching
providers.

**A background process maintains the files.** A background review agent
examines conversations and updates the memory files with new information.
It reads existing entries first and only adds genuinely new, non-obvious
facts. Outdated entries get replaced.

Review behavior differs between session types:

- **Named sessions** (e.g., `default`) run indefinitely. Reviews are
  triggered mid-session based on turn and iteration thresholds.
- **Unnamed (topic) sessions** are short-lived. By default, review only
  happens when the session completes (via `/done`), avoiding unnecessary
  mid-session spawns. This can be changed with `unnamed_review: continuous`.

Regardless of session type, a final review is attempted on completion if
thresholds were met but a review was skipped (e.g., due to concurrency).

Only one review agent runs at a time — if a review is already active,
the spawn is skipped and thresholds must be met again on the next event.
Each review agent has a hard execution timeout (default 10 minutes).

**Agents can add memories directly.** The `memory_add` and `memory_remove`
tools let agents record facts immediately without waiting for the background
review. This is useful for capturing user preferences in the moment (e.g.,
"I don't like purple, let's use green"). The tools handle path resolution,
directory creation, deduplication, and character limits automatically.

- `memory_add(key, content)` — add a fact to `"user"` (global) or `"project"` (per-project) memory
- `memory_remove(key, content)` — remove entries matching a substring

**The review agent inherits the parent session's project directory.** The
background-review plugin spawns the review agent with its working directory
set to the parent session's project directory (not the workspace root). It
computes sandbox-writable paths so the agent can write to both the
workspace-level `.nocturnal/` (for USER.md, via an absolute path in
`writes_restricted`) and the project-level `.nocturnal/memory/` (for MEMORY.md,
relative to the agent's cwd). This ensures the sandbox permits writes to both
locations without requiring the agent to escape its project directory.

**Only your main sessions get memory.** Subagents, cron jobs, and other
ephemeral sessions don't receive memory. This prevents their narrow context
from corrupting your user profile.

## Configuration

The workspace is configured in the server config at `~/.nocturnal/config.yaml`:

```yaml
server:
  workspace: "prog"    # Relative to $HOME. Defaults to $HOME if not set
```

The background review plugin is configured at
`~/.nocturnal/plugins/background-review/plugin.yaml`:

```yaml
# ~/.nocturnal/plugins/background-review/plugin.yaml
enabled: true                      # Set false to disable all functionality
mode: "memory+review"              # disabled | memory | memory+review | review
turn_threshold: 5                   # Turns before a review is triggered (named sessions)
iterations_threshold: 15            # Tool loop iterations before a review (named sessions)
agent: "memory-review"              # Internal agent used for reviews
min_interval_secs: 300              # Minimum 5 minutes between reviews
max_execution_secs: 600             # Hard timeout per review agent (seconds)
unnamed_review: completion          # completion | continuous (for unnamed/UUID sessions)
```

### Modes

| Mode | Memory tools | Background review |
|------|-------------|-------------------|
| `disabled` | No | No |
| `memory` | Yes | No |
| `memory+review` | Yes | Yes |
| `review` | No | Yes |

The plugin is enabled by default in `memory+review` mode. Set `enabled: false`
or `mode: disabled` to turn it off.

### unnamed_review

Controls when review agents are spawned for unnamed (UUID) sessions:

| Value | Behavior |
|-------|----------|
| `completion` (default) | Review only when the session completes |
| `continuous` | Use turn/iterations thresholds mid-session, like named sessions |

Named sessions always use continuous mode regardless of this setting.

The `workspace` field in the server config (`~/.nocturnal/config.yaml`) tells the
system where the workspace root is (relative to `$HOME`). Defaults to `$HOME`
if not set. This is the single source of truth used by both the server and
the background-review plugin. It is used to:
- Locate `USER.md` for the system prompt addendum
- Set the review agent's working directory
- Compute the project-relative path for sandbox `writes_restricted`

## Editing manually

You can edit the memory files directly at any time. Use your preferred editor:

```bash
$EDITOR <workspace>/.nocturnal/USER.md
$EDITOR <project>/.nocturnal/memory/MEMORY.md
```

Changes appear the next time a session starts or the conversation is compacted.
