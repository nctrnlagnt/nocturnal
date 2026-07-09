# Voice Dictation

Push-to-talk speech-to-text in the composer. Speak into your microphone
and the transcribed text streams live into the input area at your cursor.

## How to start recording

### TUI

Press any one of:

| Key             | Notes                                      |
|-----------------|--------------------------------------------|
| `F8`            | Primary binding. Works on every terminal.  |
| `Ctrl+X v`      | Leader-key variant (press `Ctrl+X`, then `v`). |
| `Ctrl+Space`    | Works in terminals that deliver it (Kitty does; some others don't). |
| `/voice` + Enter | Slash-command toggle from the composer.   |

A red dot and "Recording…" appear in the input area.

### Desktop

Press `Ctrl+D` (or `Cmd+D` on macOS). The mic button in the composer
toggles too.

## How to stop

Press the same key again, or `Escape`. The status line briefly shows
a "Finalizing…" (desktop) or "Transcribing…" (TUI) message as the end
of the stream is processed, then the final text appears at your cursor.

Text streams live into the composer as you speak, not all at once at
the end. End-of-utterance boundaries surface as newlines so partial
phrases stay on their own line and you can see them land.

## First-time setup

The first time you trigger dictation, the required speech-to-text
runtime is downloaded and installed automatically (TUI prompts with
"press i to install"; desktop shows an Install button in a popover).
This only happens once. Progress streams live into the status bar.

If the auto-install fails or you want to run it manually, run the
bundled script:

```
~/.nocturnal/scripts/install-parakeet.sh
```

## Choosing an audio device

By default, the OS default input is used. To force a specific device,
set `NOCTURNAL_VOICE_DEVICE` before launching the client:

**Linux (PulseAudio / PipeWire)**

```
NOCTURNAL_VOICE_DEVICE=micstream_virtual.monitor nocturnal
```

Find source names with `pactl list short sources`. The virtual monitor
sources (ending in `.monitor`) capture application audio — useful if
you're piping system output through Sunshine or similar.

**macOS**

```
NOCTURNAL_VOICE_DEVICE=0 nocturnal-desktop
```

macOS uses AVFoundation device indices (not names). `0` is usually the
default microphone. Run `ffmpeg -f avfoundation -list_devices true -i ""`
to enumerate.

The same env var is respected by both the TUI and the desktop app.

## Known limitations

- **English-only** by default. Multilingual models exist but are not
  shipped — swapping the model file is a manual step.
- **macOS** uses AVFoundation device indices rather than friendly names.
  There is no UI to remap them — set `NOCTURNAL_VOICE_DEVICE` to the desired
  index.

## Troubleshooting

**Mic button does nothing / "No speech-to-text backend found"**

In the TUI, press any voice key (`F8`, `Ctrl+Space`, `/voice`) — the
status line will offer to install the backend. Press `i` to run the
install script; the script's progress streams into the status bar. On
success, press your voice key again to record.

In the desktop, click the Install button in the voice popover.

**Silent recordings (no audio captured)**

Check `~/.nocturnal/logs/nocturnal-voice.log` for the child's stderr (mic device
errors, audio backend connection failures, etc.). On Linux, list your
sources with `pactl list sources` and confirm one has an active level
meter, then set `NOCTURNAL_VOICE_DEVICE` to that source. On macOS, run
`ffmpeg -f avfoundation -list_devices true -i ""` and set
`NOCTURNAL_VOICE_DEVICE` to the index of the input you want.

To confirm your mic works at all:

```
arecord -d 3 /tmp/test.wav && aplay /tmp/test.wav
```