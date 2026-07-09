# Desktop App

## What

Nocturnal ships a native desktop application for Linux and macOS in addition
to the terminal UI. It is a graphical shell over the same agent backend —
sessions, messages, providers, voice dictation, the lot — wrapped in a
multi-pane window with resizable panels, a command palette, and a rich
chat view.

The desktop does not run its own server. It locates the `nocturnal` binary
on `PATH` (or spawns one if the server is not already running) and talks to
it over the same protocol the TUI uses. Installing the desktop therefore
implies installing the CLI.

## How it works

**Same backend, different surface.** Every session, message, and tool call
you see in the desktop is produced by the same server the TUI uses.
The desktop is a thin client; it renders UI, sends RPCs, and receives
streaming events. Nothing is duplicated.

**System webview, not Electron.** The window is rendered by your OS's
native webview (WebKit on macOS, WebKitGTK on Linux). The bundle is a
single native executable with no embedded Chromium and no JavaScript
runtime.

**Real streaming.** The desktop subscribes to the server's event stream
exactly like the TUI does. Reasoning, text, and tool-call deltas arrive
incrementally and render as they come in; the agent's "thinking"
disclosure auto-opens while a response is streaming and closes when it
finishes.

**Voice dictation.** A push-to-talk mic button in the composer captures
audio and runs the same local parakeet.cpp backend the TUI uses. The
first time you click it on a fresh install, the desktop offers a one-click
installer; the streaming model downloads in the background and dictation
starts working as soon as it lands.

## Configuration

### Server discovery

The desktop connects to the server in this order:

1. An already-running `nocturnal serve` instance on the default Unix
   socket (`$NOCTURNAL_HOME/secrets/socket`).
2. If none is running, it spawns `nocturnal serve` itself.

If the connection fails on startup, the desktop shows a **Gateway
Connection Failed** overlay with a Retry button. The same overlay
covers the case where the binary is missing from `PATH`.

### Working directory

The desktop tracks a current working directory. All session, file browser,
and file operations are scoped to that directory. Switch it from the
sidebar's project picker. The default is the directory the desktop was
launched from.

### Theme

The desktop has its own light/dark theme that follows the OS appearance
by default. Override per-system from the title bar's appearance toggle.
This theme is independent of the TUI's themes — the two clients are
configured separately.

## Usage

### First launch

1. Install the CLI: `curl -fsSL https://nocturnal-agent.dev/install.sh | bash`
2. Install the desktop: re-run the same installer (it detects a fresh
   desktop install) or open the desktop archive you downloaded from the
   GitHub release.
3. Launch the desktop app. If no working provider is configured, an
   **onboarding banner** persists at the top of the window until setup
   is complete; clicking its CTA opens Settings → Providers.
4. Start chatting. The first message creates a session in the current
   directory.

### Main areas

The window has three zones:

- **Sidebar (left):** session list — pinned, recent, archived. Create,
  rename, pin, archive, delete from the context menu. The session
  picker is searchable.
- **Main content (centre):** chat thread, or full-page views for Skills,
  Profiles, Artifacts, etc.
- **File browser (right, opens a panel):** browse the working directory
  and open files for reading or editing. Clicking a file opens it in a
  file-editor overlay over the main view.

A status bar at the bottom shows gateway connection state, the agent
activity indicator, current model, context usage, and a running timer
during a stream.

### Command palette

`Ctrl+P` (or `Cmd+P` on macOS) opens a searchable command palette.
Type to filter; arrow keys to navigate; `Enter` to invoke. Works for
session operations, navigation, theme toggles, and any registered
slash command.

### Voice dictation

Click the mic in the composer (or press `Ctrl+D` / `Cmd+D`) to start
recording. Text streams live into the composer as you speak — partial
phrases land on their own line. Press the key again (or `Escape`) to
stop. The first click on a fresh install opens the parakeet installer.

### Self-update

The server runs a background task that polls the release server every
six hours. When a newer version is detected, the server broadcasts an
`UpdateAvailable` event; the desktop shows an **Update Available** modal
with an **Update Now** button. Clicking it delegates to the same
`install.sh` flow the CLI uses — the binary and the desktop bundle are
both replaced atomically and the desktop relaunches.

If the server was started more than six hours ago and the release
server has been polled since, the desktop receives the notification as
soon as it connects.

## Differences from the TUI

| | TUI | Desktop |
|---|---|---|
| Interface | Terminal | Graphical window |
| Mouse | Limited (click filenames, scroll) | Full (drag, resize, context menus) |
| Themes | Dark, Monokai, Dracula, Nord, Tokyonight-Storm, Catppuccin, Catppuccin-Macchiato, Catppuccin-Frappe, System, Clawed | Light, Dark, System |
| Command entry | Slash commands and key bindings | Command palette + slash commands |
| Voice dictation | `F8`, `Ctrl+X v`, `Ctrl+Space`, `/voice` | Mic button, `Ctrl+D` / `Cmd+D` |
| Image rendering | ANSI half-blocks (Kitty graphics protocol when supported) | Native image view in chat |

Both clients speak the same protocol and can connect to the same server.
You can run the desktop and a TUI in parallel against one server and
watch updates land in either interface.

When the desktop is launched separately from the server, ensure both
see the same `NOCTURNAL_HOME` — the socket path is resolved from
`NOCTURNAL_HOME` independently by each process.