# Upgrade and Revert

## What

Nocturnal can update itself in place without losing sessions or
configuration. The flow is the same whether you trigger it from the TUI
or the desktop — a single installer script replaces the binary, refreshes
the built-in agents and plugin manifests, and replaces the desktop bundle
if one is installed. The previous version is backed up so a single
command can roll back.

## How it works

**`/upgrade`** downloads the latest `install.sh` from the release host
and runs it with `--no-shutdown --no-modify-path`. The installer:

1. Backs up the current `nocturnal` binary, the `~/.nocturnal/agents.builtin/`
   directory, and any installed plugin manifests to
   `~/.nocturnal/backups/latest/`. The destination of each backed-up file is
   recorded in a manifest alongside the backup.
2. Downloads the new release archive, verifies the SHA-256 checksum,
   and extracts it over the existing install.
3. Refreshes `~/.nocturnal/agents.builtin/` and plugin manifests so updated
   definitions take effect.

When the installer finishes, the server broadcasts an "Update complete"
out-of-band message, then shuts itself down gracefully. The main entry
point then spawns a fresh `nocturnal serve` subprocess running the new
binary — same socket, same port args. Sessions drain cleanly, the
listener closes, and connected clients reconnect to the new process
without intervention.

**`/upgrade revert`** rolls the previous upgrade back. It uses the
backup created during the most recent `/upgrade`. If no backup exists,
the command fails with an error.

A second `/upgrade` (or `/upgrade revert`) while one is in flight is
rejected — only one upgrade runs at a time.

## Usage

### From the TUI

```
/upgrade            # download and install the latest release
/upgrade revert     # restore the previous binary and config
```

The TUI shows an out-of-band banner ("Upgrade started" / "Update
complete — restarting. Please reconnect.") and reconnects
automatically.

### From the desktop

The desktop checks for new releases at launch. When one is available, a
banner appears in the title bar with an **Update Now** button. Clicking
it delegates to the same installer. When the upgrade finishes, the
desktop relaunches itself.

### From the shell

You can run the installer manually any time:

```bash
curl -fsSL https://nocturnal-agent.dev/install.sh | bash
```

A manual run behaves the same way as `/upgrade`, including the
self-replace-and-restart flow.

## Backups

Each `/upgrade` writes a fresh backup to `~/.nocturnal/backups/latest/`. The
backup directory contains:

- `manifest` — one line per backed-up file (the destination path)
- `backed_up_from` — the version string of the backed-up install
- `nocturnal` — the previous CLI binary
- `nocturnal-desktop` (Linux) or `Nocturnal.app/` (macOS) — the
  previous desktop bundle, if one was installed
- `agents.builtin/` — the previous built-in agent definitions
- `plugins/<name>/...` — any plugin manifests that were replaced
- `install.sh` — the installer that created this backup

The backup is overwritten by the next upgrade. There is no historical
chain of backups — `/upgrade revert` always restores the most recent
previous version.

## Configuration

No configuration is required. Upgrade behaviour is automatic:

- The release URL is compiled into the binary. Default:
  `https://nocturnal-agent.dev/releases`.
- The version to install is `latest` by default. To pin a specific
  version, set `NOCTURNAL_VERSION=<version>` in the environment before
  triggering the upgrade.
- To upgrade from a custom release host (for example, a self-hosted
  mirror), set `RELEASE_BASE=<base-url>`.

| Environment variable | Default | Effect |
|---|---|---|
| `RELEASE_BASE` | `https://nocturnal-agent.dev/releases` | Base URL for release tarballs |
| `NOCTURNAL_VERSION` | `latest` | Version to install |

These variables are also respected by `install.sh` when run manually.

## Limitations

- **Same-machine only.** The installer updates the local filesystem.
  If your TUI is on one machine and the server is on another (HTTP
  mode), each machine must be upgraded independently.
- **Desktop cannot connect to a remote server over Unix socket.** The
  desktop always talks to a local server, so the desktop host is always
  the same as the server host and the single upgrade is enough.
- **One upgrade at a time.** Concurrent invocations are rejected. The
  TUI and the desktop share the same server, so there is no benefit to
  triggering twice.

If `/upgrade` fails partway through (download error, interrupted
script), re-run it. The server is still running the old binary, the
backup is in place, and the next attempt will retry cleanly.