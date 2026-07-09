# Session Storage

## What

Sessions are stored as plain files on disk — no database required. Each session
is a directory containing a metadata file and a message log. The format is
simple, fast, and easy to inspect or back up.

## How it works

Sessions live at `~/.nocturnal/sessions/by-uuid/<uuid>`. Each session directory
contains:

- **`.meta.json`** — metadata: title, model, timestamps, alias info, pause
  state, token counts
- **JSONL file** — the message log, one JSON object per line

JSONL (JSON Lines) means each line is a self-contained JSON object. This
append-only format is efficient to write — new messages are just appended to the
end of the file. No rewriting, no locking overhead, no database to maintain.

## Why this format

- **No database dependency.** No SQLite, no Postgres, no migration scripts.
  Just files.
- **Fast writes.** Appending a line to a file is about as fast as it gets.
- **Easy to inspect.** `cat`, `jq`, `grep` — standard Unix tools work on
  session files directly.
- **Easy to back up.** Copy the directory. Done.
- **Easy to clean up.** Delete a session directory to remove it.

## Browsing sessions

You can browse sessions directly on the filesystem:

```bash
# List all sessions
ls ~/.nocturnal/sessions/by-uuid/

# Read a session's metadata
cat ~/.nocturnal/sessions/by-uuid/<uuid>/.meta.json | jq .

# Search messages for a keyword
grep "keyword" ~/.nocturnal/sessions/by-uuid/<uuid>/*.jsonl
```

The TUI loads sessions on demand — only the messages needed to fill your
viewport are read. Scrolling up loads earlier messages. This keeps session
switching fast regardless of conversation length.
