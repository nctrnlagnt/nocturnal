# Computer Use

Computer use lets a Nocturnal agent observe and control the local
desktop — screenshots, accessibility trees, clicks, typing, scrolling —
through one ordinary tool named `computer_use`.

It is a general agent capability. The [ambient desktop
companion](./desktop-companion.md) can use it, and so can any other agent
you grant it to. The companion does not own computer use, and computer
use does not require the companion.

## What it is for

- “What is on my screen?” with structured window state, not only pixels
- “Click the export button and confirm the dialog”
- “Type this into the focused field”
- Short observe → act → verify loops while you watch

It is **not** a substitute for the filesystem tools, the shell, or a CI
runner. Prefer those when the job is about files and commands rather than
GUI state.

It is also not a sandboxed remote desktop. Actions run with **your**
logged-in GUI authority on the machine where the server and desktop
session live.

## Safety model

Computer use is fail-closed:

| Mode | Effect |
|---|---|
| `disabled` (default) | The tool is not advertised. Agents cannot call it. |
| `observe` | Read-only inspection (window state, screenshots, zoom, etc.). No input injection. |
| `full` | Observation plus input (click, type, key, scroll, …). After mutating window-targeted actions, a fresh screenshot is captured for the next model turn. |

Additional gates:

- An agent only sees the tool if its definition lists `computer_use`.
- Destructive key combos can be hard-blocked by the backend.
- The tool runs **out of process** from the LLM server. A hang in GUI
  code must not take down other sessions.
- Filesystem sandboxing does **not** contain GUI actions. Treat `full`
  mode like handing the agent your keyboard and mouse.

Enable it only on machines and accounts where that risk is acceptable.

## Build-time note: CUA fork

The Linux build pins four CUA crates (`cua-driver-core`,
`platform-linux`, `cursor-overlay`, `platform-macos`) to a fork at
`github.com/nctrnlagnt/cua` until the upstream `trycua/cua`
fractional-scale fix merges. The pin is at the workspace
`[patch]` table in `Cargo.toml`; the `nocturnal-computer`
`[dependencies]` table references the same rev. If you build the
CLI / desktop on Linux you are pulling from that fork. Switching
back to upstream is a one-line `[patch]` removal after the fix
lands.

## Setup

### 1. Configure the plugin

The bundled plugin lives at
`~/.nocturnal/plugins/computer-use/` (shipped from
`nocturnal-home/plugins/computer-use/`).

Copy the example config if you do not have one yet:

```bash
cp ~/.nocturnal/plugins/computer-use/config.example.yaml \
   ~/.nocturnal/plugins/computer-use/config.yaml
```

Set the mode:

```yaml
# ~/.nocturnal/plugins/computer-use/config.yaml
mode: observe   # or: full  |  disabled
```

Changes apply when the plugin restarts (restart the server or reload
plugins).

### 2. Grant the tool to an agent

Built-in `desktop-companion` already lists `computer_use`. For other
agents, add it to the agent frontmatter `tools:` list, for example:

```yaml
tools: [computer_use, read, write, edit, sessions_spawn]
```

An agent without that grant cannot call the tool even if the plugin mode
is `full`.

### 3. Platform permissions

**macOS:** Screen Recording and Accessibility must be allowed for the
Nocturnal app (and any helper the signed bundle uses). Microphone is not
required for computer use itself.

**Linux:** X11 remains the most complete path. On Wayland sessions
(`WAYLAND_DISPLAY` set), `nocturnal computer` enables CUA's native
Wayland backend automatically (`CUA_DRIVER_RS_ENABLE_WAYLAND=1`) so
screenshots and window listing do not fall through to broken X11
`GetImage` / empty `_NET_CLIENT_LIST` paths. Set
`CUA_DRIVER_RS_ENABLE_WAYLAND=0` to force the X11/XWayland path off.

**Clicks on KDE/GNOME Wayland** need CUA's `portal-input` feature
(libei + `xdg-desktop-portal` RemoteDesktop). That is enabled in our
Linux build. The first input action may show a system permission
prompt — grant it once. Requires `xdg-desktop-portal` and a desktop
backend (`xdg-desktop-portal-kde` or `-gnome`). Without portal input,
a11y trees still work but clicks fail or time out.

## How agents should use it

The tool is a single umbrella with an `action` field. The exact action
list comes from the active backend so it stays honest to what is
installed.

Recommended reliability order:

1. **`get_window_state`** (or equivalent) for the accessibility tree plus
   screenshot of the relevant window.
2. Act with **element tokens / indices** from that state when possible.
3. Fall back to **screenshot coordinates** only for non-accessible
   canvases or custom-drawn UI.
4. **`zoom`** (when available) before clicking visually ambiguous targets.
5. After every mutation, **inspect again** before the next action.

The companion and other agents are instructed not to invent private
tag protocols (`[POINT:…]`, `[[click]]`, and friends). If the tool cannot
do something, the agent should say so.

### Coordinate spaces (`coord_space`)

`click` / `drag` / `move_cursor` accept an optional `coord_space`:

| Value | `x,y` mean | Also pass |
|-------|------------|-----------|
| `image` (default, alias `window`) | Window-local pixels for `window_id` (same units as the screenshot you saw) | `pid`, `window_id` |
| `screen` | True desktop pixels (virtual desktop top-left origin) | — (omit window target) |
| `screenshot` | Pixels in an Ask Desktop / encoded monitor image | `origin_*`, `logical_*`, `encoded_*` from that screen's mapping |

`screenshot` is the image-relative path: the model points into the image it was
given; the bridge maps to screen before CUA glides and clicks.

`image` and `window` are interchangeable; both keep `window_id` semantics.
The bridge default is `image` for implementation simplicity; agents can
spell it `window` for clarity without behavioral change.

Note: on some Linux toolkits the agent-cursor glide for AX clicks aims at
AT-SPI reported bounds, which can disagree with the visible widget even when
the accessibility action still hits the right control.

### Agent cursor overlay

The tool exposes CUA's simulated cursor actions (`move_cursor`,
`set_agent_cursor_enabled`, and related style/state helpers) subject to
access mode and CUA's own `read_only` flags. Coordinate `click` / `drag`
already glide the agent cursor via CUA's built-in overlay path — no
extra `move_cursor` is required for ordinary clicks. On X11/macOS,
`move_cursor` drives the overlay only; on Linux Wayland with
virtual-pointer enabled, CUA may also warp the real pointer.

## Relationship to Ask Desktop

Ask Desktop already attaches release-time multi-monitor screenshots to
the user turn. That is a **one-shot snapshot** for the spoken
conversation.

Computer use is different:

- It is a **tool loop** the model can call multiple times in one turn.
- It can re-capture after actions to verify what changed.
- It can target window state and accessibility, not only full-monitor
  pixels.
- Spawned background agents do not keep receiving Ask Desktop
  screenshots; if they need GUI context, they call `computer_use`
  themselves (when granted).

## Same-machine assumption

Computer use assumes the Nocturnal server and the GUI being controlled
are on the **same machine**. Controlling a desktop across a remote
client/server split is intentionally out of scope: screenshots on one
host and clicks on another would be the wrong product.

Ordinary remote clients (TUI or desktop UI on another machine) can still
*view* sessions; they should not expect to project computer use onto a
machine they are not running on.

## Limitations

- Default is off; nothing GUI-related runs until you opt in.
- Not a full VM/QA lab. Background or containerized computer use may
  arrive later behind the same tool name; it is not required for the
  companion experience.
- Coordinate-only clicking is brittle on dynamic UIs — prefer
  accessibility paths.
- Multi-user or locked-screen environments are unsupported.
- The backend action set can differ slightly by OS; the advertised schema
  is what the agent is allowed to call.

## Troubleshooting

**Agent says it has no desktop tool**

Check plugin `mode` is not `disabled`, the plugin is running, and the
agent definition includes `computer_use`.

**Calls fail immediately on macOS**

Re-check Screen Recording and Accessibility for the signed Nocturnal
app. After permission changes, restart the desktop/server processes that
hold the old TCC state.

**Clicks miss or hit the wrong window**

Prefer accessibility element actions over raw coordinates. Ensure the
target window is visible and not on another virtual desktop the backend
cannot reach.

**Server stays healthy but computer use wedges**

That isolation is intentional. Restarting the computer-use component /
plugin should not require killing unrelated sessions. Do not kill a
shared `nocturnal-llm` process to unstick GUI automation.

## Related

- [Desktop Companion](./desktop-companion.md) — voice + screenshot UX that
  can call this tool
- [Desktop App](./desktop.md)
- [Plugins](./plugins.md)
- [Sandboxing](./sandboxing.md) — filesystem sandbox does not contain GUI
  actions
