# Sessions

## What

Every conversation in Nocturnal is a session. Sessions are scoped by project
directory, appear in a live-updating list in the TUI, and can be marked as
completed to keep the list clean.

## How it works

**Scoped by project directory.** When you start a session from a project
directory, it is associated with that project. The session list in the TUI
shows sessions for the current project by default. This means your work on
different projects stays organized without manual grouping.

**Live session list.** The session list updates in real time. New sessions
appear as they're created, subagent sessions show their current status, and
completed sessions can be hidden. You don't need to refresh or reload.

**Auto-titling.** Sessions get a title automatically after the first exchange.
The title is generated using the conversation context already available -- no
extra LLM request is made. This saves quota on request-billed providers (no
separate round trip) and keeps costs down on token-billed providers. You can
also set a custom title with `/title My Title`.

**Completing sessions.** Type `/done` or `/complete` in the input to mark a
session as completed. Completed sessions are hidden from the default session
list, reducing clutter. They still exist on disk and can be accessed if needed.

**Pinning sessions.** Type `/pin` to mark a session you want to return to.
Pinned sessions appear in a `Pinned` section at the top of the session list,
making them easy to find without searching. Pinning is independent of the
completed state — a session can be both pinned and completed. Use `/unpin`
to remove the pin.

**Model favorites.** You can mark frequently-used models as favorites in the
model picker (Ctrl+F). Favorites are persisted to `~/.nocturnal/tui.conf` and appear
at the top of the model list for quick access.

**Keyboard navigation.** Sessions can be navigated entirely from the keyboard.
The session picker lets you browse, filter, and switch sessions without leaving
the keyboard.

## Configuration

### TUI configuration

Favorites, theme, and other TUI preferences are stored in `~/.nocturnal/tui.conf`
(JSON format). This file is managed by the TUI itself -- you don't need to
edit it by hand.

### Session storage

Sessions live in `~/.nocturnal/sessions/by-uuid/<uuid>/`. Each session contains:

- JSONL files with the conversation history (one per subsession)
- `.meta.json` with metadata (title, token counts, status, timestamps)

Named sessions (those created with a name) also have a symlink under
`~/.nocturnal/sessions/by-name/` for easy reference.

## Usage

### Starting a new session

```bash
nocturnal              # new session in current directory
nocturnal --new        # explicitly new session
nocturnal --resume ID  # resume a specific session
```

### Session commands

| Command | Description |
|---------|-------------|
| `/done` or `/complete` | Mark session as completed (hides from list) |
| `/pin` | Pin the current session (shows in Pinned section) |
| `/unpin` | Unpin the current session |
| `/title` | Auto-generate a title (uses current context, no extra request) |
| `/title My Title` | Set a custom title |
| `/reload` | Reload config and API keys without restarting |

### Keyboard shortcuts

Use the session picker to browse sessions. In the model picker, press Ctrl+F
to toggle the current model as a favorite. In the session picker, press
Ctrl+P to toggle the pinned state of the selected session.
