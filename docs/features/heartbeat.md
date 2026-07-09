# Heartbeat — Periodic Agent Check-Ins

## What

Heartbeat mode sends periodic nudges to an idle session, prompting the agent to
continue working even when there's no user input. This is useful for long-running
tasks where you want the agent to check in at regular intervals.

## How it works

When heartbeat is enabled, a background timer wakes up at the configured
interval and injects a system message into the session. The agent sees this as
a nudge to continue if there's still work to do. The nudge includes the current
timestamp.

Heartbeat state is persisted per-session. If the server restarts, heartbeat
timers are restored for sessions that had them enabled. Timers stop
automatically when a session completes, is interrupted, or encounters an error.

## Configuration

No configuration file is needed. Heartbeat is controlled entirely through the
`/heartbeat` slash command in the TUI.

The default interval is **30 minutes**. You can set a custom interval between
1 and 1440 minutes (24 hours).

## Usage

The `/heartbeat` command accepts the following arguments:

| Command | Effect |
|---------|--------|
| `/heartbeat` | Toggle: enable if off, disable if on |
| `/heartbeat on` | Enable with default interval (30 min) |
| `/heartbeat off` | Disable |
| `/heartbeat 15` | Enable with 15-minute interval |
| `/heartbeat 5m` | Enable with 5-minute interval |

Examples:

Enable heartbeat with the default interval:

```
/heartbeat
```

Set a custom 10-minute interval:

```
/heartbeat 10
```

Disable heartbeat:

```
/heartbeat off
```

### When to use it

- **Long builds or tests.** The agent can monitor progress and report back
  periodically without you needing to prompt it.
- **Multi-step tasks.** Keep the agent moving through a checklist without
  manual intervention.
- **Background monitoring.** Have the agent check on a deployed service at
  regular intervals and report anomalies.
