# Ambient Desktop Companion

The ambient desktop companion turns Nocturnal into an ambient screenshot-aware layer
on your machine: hold a global hotkey, speak, release, and get a spoken
answer about whatever is on screen — without switching into a chat
window first.

It is part of the desktop app, not a separate product. The same server,
sessions, agents, and tools power it.

## What you get

Two global push-to-talk shortcuts:

- **Ask Desktop** — hold, speak a question or request, release. Nocturnal
  captures one screenshot per monitor, sends the transcript plus those
  images to a dedicated desktop session, speaks the answer, and can act
  on the desktop when computer use is enabled.
- **Dictate** — hold, speak, release. The transcript is pasted into
  whatever app was focused when you pressed the key. No screenshot is
  taken.

Around those shortcuts:

- A **system tray** entry so the desktop can stay resident with no chat
  window open.
- A small **status pill** that shows listening / dictating / thinking /
  speaking without stealing keyboard focus.
- A persistent **Desktop Session** (`main:desktop`) you can open any time
  to read the full conversation, type a follow-up, or inspect tool use.
- **Spoken replies** through local text-to-speech (Pocket TTS when
  installed, otherwise the OS speech engine).
- Optional **computer use** so the companion can inspect and control the
  desktop through the normal `computer_use` tool — see
  [Computer Use](./computer-use.md).

Composer mic dictation inside a chat window is a different feature. That
path is documented in [Voice Dictation](./voice-dictation.md). Dictate
here is system-wide and targets the foreground application.

## How Ask Desktop works

1. Hold the Ask Desktop shortcut (default `Ctrl+Shift+Space`, or
   `Cmd+Shift+Space` on macOS).
2. Speak. The status pill shows that it is listening.
3. Release. Nocturnal finalizes the transcript and takes **one screenshot
   per connected monitor**.
4. Those images and your words go to the `main:desktop` session as a
   single turn. Screen-mapping details stay hidden in the chat UI; you
   see the transcript and the screenshots.
5. The companion answers out loud. Long work can be handed to a
   background agent the same way it would from any other session.

Nothing is captured while the app is merely sitting in the tray. Capture
happens only on a normal Ask Desktop release — not on key-down, not while
you hold the key, not on cancel, and not continuously in the background.

If any monitor fails to capture, the turn is aborted rather than sending
a partial set of screenshots.

## How Dictate works

1. Focus the app you want to type into.
2. Hold Dictate (default `Ctrl+Shift+D` / `Cmd+Shift+D`).
3. Speak, then release.
4. Nocturnal pastes the finished transcript into the app that was focused
   when you pressed the key.

Dictate records the target at key-down. If that window is gone or focus
moved to a different app before paste, insertion is skipped instead of
dumping text somewhere unexpected. Your previous clipboard is restored
when it is safe to do so (only if it still holds the dictated text).

Dictate never screenshots the desktop and never opens the Desktop
Session.

## Desktop Session

Ask Desktop always talks to the named session `main:desktop`, using the
built-in `desktop-companion` agent.

- Voice turns and typed turns share one history.
- **Open Desktop Session** from the tray focuses an existing window or
  opens one.
- You can scroll back, copy text, attach files, or type follow-ups like
  any other chat.
- Spawning background agents (“research this”, “implement that”) uses the
  normal session tools. Those agents do **not** keep receiving live
  screenshots; they work from the conversation and their own tools.

The companion agent is tuned for speech: short answers, no markdown
theater, and tolerance for speech-to-text mistakes. It treats every user
message as dictated audio, not carefully typed prose.

## Status pill

While companion mode is active, a small pill can sit on screen and mirror
the current phase:

| Phase | Meaning |
|---|---|
| Idle | Resident, waiting for a shortcut |
| Listening | Ask Desktop is recording |
| Dictating | Dictate is recording |
| Thinking | Waiting on the model / tools |
| Speaking | Playing the spoken reply |

The pill is meant to stay out of your way: non-activating, not a normal
taskbar window, and click-through except where an explicit control (such
as expand/collapse) is provided. On Linux it is implemented for X11;
Wayland and polished macOS panel behavior continue to improve with the
desktop shell.

## Tray menu

With companion mode enabled, the desktop can hide its last window instead
of quitting. The tray menu provides:

- **Open Desktop Session** — show `main:desktop`
- **Pause Shortcuts** / resume — temporarily disable global hotkeys and
  stop any in-flight speech
- **Settings** — companion shortcuts and related options
- **Quit** — unregister hotkeys, stop recording/speech, exit fully

## Settings

Companion settings live in the desktop app (tray → Settings, or the
desktop settings UI):

| Setting | Default | Purpose |
|---|---|---|
| Companion mode | on | Keep the app resident in the tray when windows close |
| Shortcuts enabled | on | Register the global hotkeys |
| Ask Desktop shortcut | `Ctrl/Cmd+Shift+Space` | Screen-aware push-to-talk |
| Dictate shortcut | `Ctrl/Cmd+Shift+D` | System-wide dictation paste |
| Emergency stop | `Ctrl/Cmd+Shift+Escape` | Cancel speech / clear in-flight companion UI work |
| Allow foreground actions | off | Legacy companion-local mutation gate; desktop control should go through [Computer Use](./computer-use.md) |
| Voice reference WAV | bundled default | Optional custom voice sample for Pocket TTS |

Shortcuts are ordinary modifier-plus-key chords. Modifier-only gestures
(for example holding Control+Option with no key) are not supported.

## Speech output

Replies are spoken automatically for Ask Desktop turns.

- **Pocket TTS** is used when the local model is installed (via the
  uniprocess `nocturnal tts` path).
- If Pocket TTS is unavailable, the desktop falls back to the **OS speech
  engine** (`speech-dispatcher` / `espeak` on Linux, system voices on
  macOS).
- New Ask Desktop turns, emergency stop, Pause Shortcuts, and Quit cancel
  playback immediately.
- Text is cleaned for speech: no heading markup, no emoji noise, no
  reading control tags aloud.

You can supply a custom reference WAV in settings to clone a voice with
Pocket TTS; leave it empty for the bundled default.

## Computer use from the companion

When you ask the companion to click, type, open, scroll, or otherwise
drive the GUI, it uses the same `computer_use` tool any other granted
agent would use. There is no secret on-screen tag language.

Computer use is **off by default** and must be enabled in the
computer-use plugin config. Details, access modes, and safety notes are
in [Computer Use](./computer-use.md).

## Permissions

### Linux

- Microphone access for STT.
- **X11 or Wayland** session. The companion picks a backend from
  `XDG_SESSION_TYPE` / `WAYLAND_DISPLAY` (same idea as other
  screenshot apps).

**X11**

- **`xdotool`** — Dictate paste, focused-window capture, and the
  geometry lookup the pill uses when it cannot reach a chat window.
- **`xclip`** — preferred clipboard owner for Dictate (`xclip
  -silent` avoids the set-then-drop race on `Ctrl+V`). Falls back
  to `arboard` if missing.
- **ImageMagick `import`** — fallback for focused-window capture on
  awkward windows (e.g. some terminals). Multi-monitor Ask Desktop
  still uses the pure-Rust xcb path without it.
- Status pill uses X11 shape + dock hints for click-through chrome.

**Wayland**

- **Ask Desktop capture** uses a host screenshot tool that already
  speaks the portal/compositor protocols: `grim` (wlroots/Sway/
  Hyprland), `spectacle -b -n` (KDE), or `gnome-screenshot`.
  First use may show a one-time screen-share grant; after that it
  should feel like any other screenshot app. Focused-window-only
  capture falls back to full desktop for now.
- **`wl-clipboard`** (`wl-copy` / `wl-paste`) — Dictate clipboard.
- **`wtype`** (preferred) or **`ydotool`** — Dictate paste into the
  focused app. `ydotool` needs uinput/`ydotoold` permissions.
- Focus tracking uses `hyprctl` / `swaymsg` when present; elsewhere
  Dictate still pastes to whatever is focused (weaker switch detection).
- Status pill uses a **GDK input region** matching the X11 tab
  silhouette so clicks outside the paint pass through. Placement is
  top-center, with Hyprland/Sway focus follow when those CLIs exist.
  Layer-shell docking (panel semantics on every compositor) is still
  improving; GNOME may keep a simpler floating always-on-top window.

### macOS

The desktop needs:

- **Microphone** — dictation and Ask Desktop (prompted on first hold)
- **Screen Recording** — Ask Desktop screenshots (prompted on first capture)
- **Accessibility** — Dictate paste into the focused app

**Companion → macOS permissions** in settings shows live On/Off status,
**Request access…**, and deep links into System Settings. Grant toggles
when prompted; the installer cannot flip them for you.

**Unsigned / dev builds:** macOS often forgets TCC grants when the binary
path or signature changes. If Ask Desktop returns a blank screen or
Dictate will not paste after a rebuild, open Privacy & Security, remove
stale **Nocturnal** rows under Screen Recording and Accessibility, then
enable the new build again. A stable signed build keeps grants across
upgrades when you have one.

The status pill stays on all Spaces (`visible_on_all_workspaces`). It is
still a simple always-on-top window (not a full NSPanel notch), but it
should remain visible while you work.

## Privacy

- Idle tray residence does not capture the screen or the mic.
- Screenshots are taken only on Ask Desktop release.
- Audio is transcribed locally; PCM does not leave the machine on the
  STT path.
- Screenshots and transcripts are sent to whatever model/provider you
  configured for the desktop-companion agent, through your normal
  Nocturnal provider setup.
- Between activations, nothing is captured or uploaded by the companion
  layer.

## Requirements

- Nocturnal desktop app (Linux or macOS).
- Working provider/model configuration (the companion agent defaults to
  your `coworker` alias).
- Local speech-to-text runtime (same parakeet install used by composer
  dictation — the desktop prompts to install on first use).
- Optional: **Pocket TTS** model (sherpa-onnx int8, ~93 MB) for
  higher-quality spoken replies. On first Ask Desktop that wants to
  speak, the daemon downloads it from k2-fsa/sherpa-onnx into
  `~/.local/share/dev.nocturnal-agent.nocturnal/pocket-tts/`. Set
  `NOCTURNAL_TTS_OFFLINE=1` to disable the download (the OS speech
  engine is then used instead).
- Optional: computer-use plugin in `observe` or `full` mode if you want
  GUI control.

## Troubleshooting

**Hotkeys do nothing**

Check that companion mode and shortcuts are enabled, and that another app
has not claimed the same chord. Try Pause Shortcuts then re-enable, or
pick different chords in settings.

**Ask Desktop answers without seeing the screen**

Confirm Screen Recording permission (macOS) and check
`$TMPDIR/nocturnal-companion.log` for capture errors. On failure the
desktop aborts the turn instead of sending a blind request.

**Dictate pastes into the wrong place**

Focus the target app *before* pressing Dictate. The target is frozen at
key-down; switching apps mid-hold will cancel a safe paste.

**No voice / robotic OS voice**

Install or repair Pocket TTS if you want the higher-quality path; otherwise
install `speech-dispatcher` or `espeak` on Linux. Check desktop logs if
playback cancels immediately.

**Companion quit when I closed the window**

Enable companion mode so the last window hides instead of exiting. Use
tray → Quit when you want a full shutdown.

**macOS permissions keep resetting**

Unsigned or differently-bundled builds look like a new app to TCC. Use
the official signed desktop bundle and avoid renaming it between
releases.

## Related

- [Desktop App](./desktop.md) — the graphical shell
- [Voice Dictation](./voice-dictation.md) — composer push-to-talk
- [Computer Use](./computer-use.md) — GUI observe/act tool
- [Subagents](./subagents.md) — background work the companion can spawn
- [Sessions](./sessions.md) — named sessions including `main:desktop`
