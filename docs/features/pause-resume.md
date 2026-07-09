# Pause and Resume

## What

The `/pause` and `/unpause` commands let you stop and resume agent sessions.
Pausing halts a session at the next safe point; unpausing picks up exactly
where it left off. Pause state survives server restarts.

## How it works

**Pause.** The `/pause` command stops a session at the next tool boundary — the
point between one tool call finishing and the next starting. This is a safe
stop point because no tool is mid-execution. The agent will not be interrupted
mid-operation.

**Resume.** The `/unpause` command resumes a paused session. The agent picks up
from exactly where it stopped, with full context of what it was doing.

**Survives restarts.** Pause state is persisted. If you pause a session and then
restart the server (or the server restarts unexpectedly), the session remains
paused. Run `/unpause` after the server is back to continue.

## Configuration

No configuration required. Pause and resume are always available.

## Usage

In the TUI, type the slash command while viewing the session you want to
control:

```
/pause     # Pause the current session
/unpause   # Resume a paused session
```

Common scenarios:

- **Inspect state.** Pause a long-running workflow to examine files, logs, or
  configuration before letting the agent continue.
- **Change configuration.** Pause, edit `config.yaml` or `.env`, run `/reload`,
  then unpause. The agent continues with the updated configuration.
- **Take a break.** Pause a session that's working on a lengthy task. It will
  wait indefinitely until you unpause it.

There is a brief delay between `/pause` and the session actually stopping — the
agent finishes its current tool call first. You will see the session status
change to paused once it reaches the tool boundary.
