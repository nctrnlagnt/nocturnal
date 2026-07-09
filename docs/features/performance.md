# Performance

## What

Nocturnal is designed to be fast and light. The server starts in ~13ms, uses
~22MB of RAM, and keeps CPU usage low even during active streaming. Long
sessions don't slow down over time.

## How it works

**Fast startup.** The server process initializes in ~13ms. There is no
heavy framework bootstrapping, no database migration, no warm-up period. The
TUI connects, loads the session list, and is ready immediately.

**Low CPU while streaming.** Active streaming uses approximately 0.7-2% CPU.
Background sessions (subagents running but not actively rendered) use negligible
CPU -- the server processes their streams but the TUI only updates summary
information.

**Low memory.** The server typically uses ~22MB of RAM regardless of how many
sessions exist or how long they've been running. Session history is stored on
disk and loaded on demand, not held in memory.

**Zero-allocation streaming rendering.** The TUI's rendering pipeline parses
streaming content once and incrementally updates the display as new chunks
arrive. It does not re-parse or re-render the entire message on each chunk.
The only case that triggers a full re-parse is a terminal width change. This
keeps the TUI responsive even with many messages on screen.

**No slowdown for lengthy sessions.** Session history is appended to JSONL
files on disk. Loading old messages does not require reading the entire file --
the server can load specific ranges. The TUI only requests enough history to
fill the viewport. Scrolling up loads more on demand.

**Efficient file-based storage.** Sessions are stored as JSONL files under
`~/.nocturnal/sessions/`. There is no database. Each session gets a directory
containing one JSONL file per subsession, plus a `.meta.json` file with
metadata (title, token counts, status) for fast listing without reading the
full history.

| Metric | Typical value |
|--------|--------------|
| Server startup | ~13ms |
| CPU while streaming | 0.7-2% |
| CPU for background sessions | Negligible |
| RAM | ~22MB |
| Session loading | Fills viewport only |

## What this means in practice

- You can run dozens of background subagents without the TUI becoming sluggish.
- You can open multiple TUIs connected to the same server without compounding
  resource usage.
- Switching between sessions is near-instant (< 100ms) because only the
  viewport contents are loaded.
- Sessions with thousands of messages perform the same as fresh sessions.
