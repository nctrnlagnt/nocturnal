# Getting Started

This page walks through installing Nocturnal, configuring a provider,
and running your first session.

## Requirements

- **Linux** (x86_64 or aarch64), **macOS** (Apple Silicon or Intel), or
  **WSL2** on Windows.
- A POSIX shell (`bash` or `zsh`) on `PATH`.
- An account with one of the supported LLM providers — see
  [Providers](providers.md) for the full list.

The installer does not need root and does not modify anything outside
your home directory by default.

## Install

### One-line installer (recommended)

```bash
curl -fsSL https://nocturnal-agent.dev/install.sh | bash
```

The installer:

1. Detects your OS and architecture.
2. Downloads the matching release archive.
3. Extracts the `nocturnal` binary into `~/.local/bin/`.
4. Writes default configuration and built-in agents under `~/.nocturnal/`.
5. Adds `~/.local/bin` to your shell's `PATH` if it is not already there
   (skipped with `--no-modify-path`).
6. Installs the desktop app on Linux and macOS unless `--tui-only` is
   passed.

Re-running the installer upgrades an existing install in place. The
previous binary and configuration are backed up to `~/.nocturnal/backups/latest/`
so a subsequent `/upgrade revert` can roll back.

### Manual install

Every release on
[GitHub Releases](https://github.com/nocturnal-agent/nocturnal/releases)
ships detached `.sha256` checksums for each archive. Download, verify,
and extract:

```bash
curl -fsSL -O https://github.com/nocturnal-agent/nocturnal/releases/latest/download/nocturnal-1.0.x-linux-x86_64.tar.gz
curl -fsSL -O https://github.com/nocturnal-agent/nocturnal/releases/latest/download/nocturnal-1.0.x-linux-x86_64.tar.gz.sha256
sha256sum -c nocturnal-1.0.x-linux-x86_64.tar.gz.sha256
tar -xzf nocturnal-1.0.x-linux-x86_64.tar.gz
mv nocturnal ~/.local/bin/
mkdir -p ~/.nocturnal
mv nocturnal-home/* ~/.nocturnal/    # default config and built-in agents
```

The `nocturnal-home/` directory inside the archive contains the default
`config.yaml`, `agents.builtin/`, and `plugins/` you need for first run.

### Verifying the install

```bash
nocturnal --version
```

The installer prints the installed version when it finishes. Run
`nocturnal` with no arguments to launch the TUI and confirm it starts.

## Configure a provider

Nocturnal needs an API key for at least one LLM provider before it can
do anything useful. Keys live in `~/.nocturnal/secrets/.env` (private to
`nocturnal serve` — never injected into child processes).

### Option 1: environment variable

If you already export `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, or
`GOOGLE_API_KEY` in your shell, Nocturnal will pick it up automatically.
The installer-created `config.yaml` already references these names.

### Option 2: the `.env` file

Add the key to `~/.nocturnal/secrets/.env`:

```bash
chmod 700 ~/.nocturnal/secrets
cat >> ~/.nocturnal/secrets/.env <<'EOF'
ANTHROPIC_API_KEY=sk-ant-...
EOF
chmod 600 ~/.nocturnal/secrets/.env
```

Then type `/reload` in the TUI (or restart the server) to pick it up.

### Option 3: `/connect` in the TUI

Launch the TUI and type `/connect`. The command walks you through
choosing a provider type, entering the base URL, and pasting your key.
The key is stored in `~/.nocturnal/secrets/.env` and the provider is appended
to `~/.nocturnal/config.yaml`.

For ChatGPT OAuth specifically, see [Providers](providers.md).

## Start your first session

```bash
cd ~/projects/my-app
nocturnal
```

You will see a session picker (or a fresh session, if there are none for
this directory). Type a message — the agent responds. Sessions are
scoped by working directory, so each project gets its own conversation
history.

### Useful first commands

| Command | What it does |
|---|---|
| `/models` | List every model from your configured providers |
| `/connect` | Add a new provider interactively |
| `/title <name>` | Rename the current session |
| `/done` | Mark the session completed (hides from the list) |
| `/reload` | Re-read `config.yaml` and `.env` without restarting |

A full list lives in [Slash Commands](slash-commands.md).

## Optional: enable a bridge

To chat with your agent from Telegram or Discord, enable the
corresponding plugin from inside the TUI:

```
/plugins telegram enable
/plugins discord enable
```

Setup steps for each platform are in [Telegram](telegram.md) and
[Discord](discord.md). Tokens go in `~/.nocturnal/secrets/.env` or via the
plugin's `/telegram` and `/discord` slash commands.

## Optional: install the desktop app

The desktop app (Linux and macOS) is included by the installer by default.
If you want a CLI/TUI-only install, pass `--tui-only`:

```bash
curl -fsSL https://nocturnal-agent.dev/install.sh | bash -s -- --tui-only
```

The desktop launches the same `nocturnal` binary it finds on `PATH` and
talks to it over the standard Unix socket. See [Desktop](desktop.md) for
the full feature set.

## Updating

Nocturnal updates itself in place. The simplest path is the in-app
command:

```
/upgrade
```

This downloads the latest installer, runs it, then restarts the server
against the new binary. Sessions survive the upgrade. If something goes
wrong, `/upgrade revert` restores the previous binary and configuration
from `~/.nocturnal/backups/latest/`. See [Upgrade](upgrade.md) for the
details.

## Uninstalling

Nocturnal is fully self-contained in three locations:

- `~/.local/bin/nocturnal` — the binary
- `~/.local/share/applications/` — desktop entry (Linux)
- `~/.nocturnal/` — state directory (config, sessions, agents, plugins, logs, backups)

To remove it:

```bash
rm ~/.local/bin/nocturnal
rm -rf ~/.nocturnal
# Linux: remove the desktop entry and icon if you installed them
rm ~/.local/share/applications/nocturnal.desktop
```

That is the complete footprint. Nothing is written to `/etc`, `/usr`,
or any system location by default.

## Troubleshooting

For installation problems, check the server log at
`~/.nocturnal/logs/nocturnal-llm.log`. If the server fails to start, the most
common causes are:

- **`PATH` not set.** `~/.local/bin` must be on your `PATH`. The
  installer adds it to `~/.bashrc` or `~/.zshrc` automatically; if you
  run a different shell, add it yourself.
- **Missing system dependency.** Bubblewrap (Linux), `sandbox-exec`
  (macOS), or `webkit2gtk` (Linux desktop) may need to be installed via
  the system package manager.
- **API key not set.** The TUI shows a clear error when no provider is
  configured. Use `/connect` to add one.

For runtime issues after a successful install, the server log is the
primary place to look. For desktop issues, the same log captures
desktop-side RPC traffic.