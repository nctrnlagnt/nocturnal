# Images

## What

You can paste images directly into conversations. Agents with vision-capable
models can see and analyze them. Images are rendered inline in the TUI at the
current terminal width. The default rendering uses the Kitty graphics
protocol for pixel-accurate output, with ANSI art as a fallback. See
[Rendering mode](#rendering-mode) below.

## How it works

**Pasting images.** When you paste an image from your clipboard (`Ctrl+V` or
`Ctrl+Shift+V`), the TUI encodes it as base64 and sends it to the server along
with your message. Supported formats: PNG, JPEG, WebP, GIF.

**Agent analysis.** The server includes the image data in the request to the
LLM provider. Vision-capable models (e.g., Claude with vision, GPT-4o) can
read the image and respond with analysis, descriptions, or answers about its
content. Non-vision models receive a text placeholder instead.

**Terminal rendering.** After processing, the server generates a thumbnail
and the TUI renders it for display at the current terminal width. When
the terminal width changes, the image is re-rendered to fit. See
[Rendering mode](#rendering-mode) for the two rendering paths and how to
switch between them.

The rendering pipeline:

1. You paste an image from clipboard
2. TUI encodes it as base64 and sends it with your message
3. Server decodes it, generates a thumbnail, and stores it
4. For vision models, the image is included in the LLM request
5. TUI renders the thumbnail at the appropriate width using the
   selected rendering mode (see [Rendering mode](#rendering-mode))
6. TUI displays it inline in the conversation

## Agent-sent images

Agents can attach images to their responses from inside their own output.
The mechanism is the `◆ media:` directive, a single line on its own that
names a local file path:

```
The chart looks like this:
◆ media: /tmp/chart.png
And the trend continues through Q3.
```

Behavior:

- The directive is parsed out of the streamed text by the server. The
  rendered user-visible text is identical to a text-only response — the
  directive line itself is stripped before display.
- The named file is treated the same as a user-pasted image: it's
  decoded, thumbnailed, and rendered for display at the recipient's
  terminal width using the rendering mode selected in their TUI
  (see [Rendering mode](#rendering-mode)).
- The directive works across every transport — TUI, Desktop, Telegram,
  and Discord. Telegram and Discord route the image to their native
  attachment channels rather than ANSI art.
- An empty path (`◆ media:` with nothing after it) is a no-op.
- The directive is stripped in the response parser alongside the
  `◆ title:` directive, so neither enters the LLM context.

The directive is independent of vision capability — agents that can't
see images can still use it to send them, the same way users can paste
images into a non-vision session.

## Rendering mode

The TUI can render images two ways. Switch between them at runtime or
set the default in config.

- **`graphics`** (default) — uses the [Kitty graphics protocol][kitty]
  to send the original image to your terminal. The terminal composites
  it directly, so you get crisp, pixel-accurate images that scroll
  naturally with the text. Falls back automatically when the terminal
  doesn't support it.
- **`ansi`** — renders the image as truecolor half-block characters.
  Works in every terminal, but at the cost of dithering artifacts on
  gradients and photos.

[kitty]: https://sw.kovidgoyal.net/kitty/graphics-protocol/

### Which mode should I use?

Start with the default (`graphics`). If your terminal supports it,
you'll get noticeably better-looking images for free. Switch to `ansi`
if:

- Your terminal claims Kitty support but renders images incorrectly
- You're running inside a remote session (SSH, tmux) where the
  graphics protocol can't pass through cleanly
- You want consistent rendering across all terminals

### How to switch

- Slash command: `/images` (toggles between `ansi` and `graphics`)
- Key binding: `Ctrl+X i`
- Command palette: open with `Ctrl+P`, search for "Images" (the entry
  is labeled `Images (graphics)` or `Images (ansi)` depending on the
  current mode), press Enter to toggle

The choice is saved immediately and restored on the next launch.

### Setting the default in config

To set the mode without opening the TUI, edit `~/.nocturnal/tui.conf`:

```json
{
  "image_mode": "graphics"
}
```

Valid values:

- `"graphics"` — use Kitty when available, fall back to ANSI otherwise (default)
- `"ansi"` — always render as ANSI art, even on terminals that support Kitty

### Terminal compatibility

| Terminal             | `graphics` mode | Notes                          |
|----------------------|-----------------|--------------------------------|
| Kitty 0.28+          | yes             | Full support                   |
| Ghostty              | yes             | Full support                   |
| WezTerm              | yes             | Supports the protocol          |
| Konsole              | yes             | Needs a patched build          |
| iTerm2               | falls back      | Uses `ansi`                    |
| Apple Terminal       | falls back      | Uses `ansi`                    |
| gnome-terminal       | falls back      | Uses `ansi`                    |
| Alacritty            | falls back      | Uses `ansi`                    |
| Anything else        | falls back      | Uses `ansi`                    |

Detection checks `$KITTY_WINDOW_ID`, `$TERM`, and `$TERM_PROGRAM`. When
detection gets it wrong, set `image_mode` to `ansi`.

### Known limitations

- **Animated images** are not supported. Each image is rendered as a
  single still frame.
- **tmux and similar multiplexers** may strip or mangle the graphics
  protocol sequences. If images stop appearing when working through
  tmux, switch to `ansi`.
- **Image appearance can vary by terminal** because each terminal
  implements the protocol slightly differently.
