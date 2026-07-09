# TUI

## What

The Nocturnal terminal interface is a full-featured client for interacting with
agents. It renders markdown in real time as messages stream in, with syntax
highlighting, responsive tables, and multiple color themes. It is designed to
stay fast even in very long sessions.

## How it works

**Markdown rendering.** Messages are rendered as structured markdown: headings,
bold, italic, code spans, bullet lists, numbered lists, blockquotes, links,
horizontal rules, and tables. Tables reflow to fit the terminal width. Code
blocks are syntax-highlighted.

**Themes.** The TUI ships with several color themes:

| Theme | Description |
|-------|-------------|
| Dark | Default dark theme |
| Monokai | Warm, high-contrast dark theme |
| Dracula | Purple-toned dark theme |
| Nord | Cool blue-gray dark theme |
| Tokyonight-Storm Dark | Deep blue dark theme |
| Catppuccin | Catppuccin dark theme |
| Catppuccin-Macchiato | Darker Catppuccin variant |
| Catppuccin-Frappe | Medium-dark Catppuccin variant |
| System | Uses your terminal's color palette |
| Clawed | High-contrast theme |

Switch themes with the command palette (`Ctrl+P`, then "theme") or `Ctrl+X`
followed by `t`.

**System theme auto-detect.** The "System" theme uses OSC 10/11 escape
sequences to detect your terminal's default foreground and background colors,
plus OSC 4 to capture the ANSI palette. Detected colors are cached in config
so they're available on next startup without re-detecting. Works in terminals
that support these sequences (Kitty, WezTerm, modern xterm).

**Select and copy.** Text in the conversation can be selected with the mouse
and copied to the clipboard. The TUI uses OSC 52 escape sequences where
supported, with fallback to platform clipboard commands (`wl-copy`, `xclip`,
`pbcopy`).

**Keybindings.** The TUI uses emacs-like keybindings for input editing and
navigation:

| Key | Action |
|-----|--------|
| `Ctrl+A` | Cursor to start of line |
| `Ctrl+E` | Cursor to end of line |
| `Ctrl+K` | Delete from cursor to end of line |
| `Ctrl+W` | Delete word backward |
| `Ctrl+U` | Half-page scroll up |
| `Ctrl+D` | Half-page scroll down |
| `Ctrl+L` | Switch to last session |
| `Ctrl+P` | Open command palette |
| `Ctrl+V` | Paste from clipboard |
| `Ctrl+J` | Insert newline in input |
| `Ctrl+X` | Leader key (followed by another key) |
| `Ctrl+X e` | Open input in `$EDITOR` |
| `Ctrl+X u` | Undo last change |
| `Ctrl+X r` | Redo change |
| `Ctrl+X x` | Export conversation to file |
| `Ctrl+X y` | Copy last assistant response |
| `Ctrl+X i` | Initialize project |
| `Ctrl+X s` | Copy session ID to clipboard |
| `Alt+B` / `Alt+F` | Move cursor back/forward one word |
| `Ctrl+C` (double-tap) | Quit |

**Click on filenames.** Clicking on a file path in a message (e.g.
`src/index.js:42`) opens it in your `$EDITOR`. The TUI detects file paths at the
click position and opens them at the correct line number.

**Click on agent IDs.** Clicking on an `agent:<uuid>` reference in a message
navigates to that subagent's session, letting you jump between parent and child
sessions quickly.

**Scroll and viewport.** The viewport is optimized for long sessions with
lazy message materialization. Switching between sessions is fast enough to
keep up with rapid keyboard navigation.

**Subagent panel.** A side panel shows all active subagents for the current
session with real-time status updates: running state, token counts, tool usage,
and the last output line. This lets you monitor what spawned agents are doing
without switching away from the current conversation.

**Palette cycling.** During message generation, the TUI animates the cursor
color using OSC 21 palette cycling sequences. This creates a visual pulse
effect without redrawing screen content. The palette is restored when
generation completes. Works in terminals that support dynamic palette
manipulation (Kitty, WezTerm).

**Welcome animation.** The welcome screen can display an animated splash
image or static ANSI art. Three modes: `animated` (plays a short movie loop),
`static` (single splash image), or `none` (text-only welcome). Toggle with
`/welcome-mode` and control loop playback with `/movie-loop`.

**File picker.** Type `@` followed by a character to open a fuzzy file search
dialog. Select a file to insert its path into the input. The picker uses
project-relative paths and ranks matches by filename priority. Refresh
the file list with `Ctrl+R`.

**Connect dialog.** The `/connect` command opens a provider management dialog.
View configured providers with their API key status, add new providers, or
edit existing ones. Supports OpenAI-compatible providers with custom base URLs.

**Slash commands.** The TUI supports local slash commands:

| Command | Description |
|---------|-------------|
| `/export` | Export conversation to timestamped file |
| `/share` | Share session (copies session ID) |
| `/init` | Initialize project for current session |
| `/status` | Show connection status |
| `/connect` | Open provider management dialog |
| `/welcome-mode` | Toggle welcome animation mode |
| `/movie-loop` | Toggle welcome movie loop playback |

## Configuration

Theme and other TUI preferences are stored in `~/.nocturnal/tui.conf` and can be
changed from within the TUI itself (no manual editing required).

The `$EDITOR` environment variable controls which editor opens when you click
a filename or press `Ctrl+X e`. Set it in your shell profile:

```bash
export EDITOR=vim
```
