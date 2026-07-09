# External Editor

## What

Open files and compose messages in your preferred editor directly from the TUI.
Click a filename shown in the conversation to open it, or use the editor to
write longer messages before sending.

## How it works

The TUI uses your `$EDITOR` environment variable to determine which editor to
launch. When you click a filename or trigger the editor for a message, the TUI
opens the file in that editor and waits for you to close it.

## Configuration

Set your preferred editor via the `EDITOR` environment variable:

```bash
export EDITOR=vim      # or nano, code, emacs, etc.
```

Add this to your `~/.bashrc` or `~/.zshrc` to make it persistent.

## Usage

### Opening files

When the agent mentions a filename (e.g., in a tool result or code block), click
on it in the TUI. The file opens in your editor at the relevant line if a line
number is available.

### Editing messages

Before sending a message, you can open your external editor to compose it. This
is useful for longer prompts, multi-line code snippets, or any input where the
TUI input field feels cramped. Write and save in your editor, and the content
is sent when you close it.
