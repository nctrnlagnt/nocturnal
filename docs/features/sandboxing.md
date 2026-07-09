# Sandboxing

## What

Tool execution runs in a separate process from the LLM interaction, enabling strong isolation. On Linux and WSL, Bubblewrap provides filesystem-level sandboxing. On macOS, the built-in `sandbox-exec` is used. Sandboxed agents cannot access your API keys or write outside allowed directories.

## How it works

**Separate process context.** The LLM server and tool execution run in different processes. The tool process is sandboxed — it has restricted filesystem access and cannot see provider API keys, even if they exist in the environment or on disk.

**Platform-specific isolation.**

| Platform | Mechanism |
|----------|-----------|
| Linux / WSL | Bubblewrap (`bwrap`) — filesystem namespace isolation |
| macOS | `sandbox-exec` — built-in macOS sandboxing |

**API key protection.** By default, the sandbox hides `~/.nocturnal/` from the tool execution process. This prevents agents from reading `~/.nocturnal/secrets/.env` or `config.yaml`, which contain API keys and other secrets.

**Write restrictions.** Even within the sandbox, agents can only write to directories you explicitly allow. This prevents accidental or malicious writes to sensitive locations.

## Configuration

Sandbox settings go in `~/.nocturnal/config.yaml`:

```yaml
sandbox:
  enabled: true
  hide_nocturnal_home: true   # prevents agents from reading ~/.nocturnal/
  allow_gpu: false       # opt in to GPU passthrough for ffmpeg hardware acceleration
```

| Setting | Default | Description |
|---------|---------|-------------|
| `enabled` | `true` | Enable or disable sandboxing entirely |
| `hide_nocturnal_home` | `true` | Hide `~/.nocturnal/` from the sandboxed process |
| `allow_gpu` | `false` | Expose GPU device nodes (`/dev/dri`, `/dev/kfd`, `/dev/nvidia*`) and read-only `/sys` to the sandbox, and allow `iokit-open` on macOS. Required for ffmpeg hardware acceleration (QSV / AMF / NVENC on Linux, VideoToolbox on macOS). Off by default because GPU device nodes are DMA-capable. |

### Write access

Control which directories agents can write to:

```yaml
project:
  allowed_write_dirs:
    - .
    - ~/.agent-tui
    - /run
```

- `.` — the project directory (always recommended)
- Additional paths as needed for your workflow

Agents can read anywhere the sandbox permits but can only write to listed directories.

## Usage

Sandboxing is enabled by default. No configuration is required for basic use — the defaults protect your API keys and restrict writes to the project directory.

### Disabling the sandbox

If you need unrestricted tool execution (not recommended for normal use):

```yaml
sandbox:
  enabled: false
```

### Allowing additional write paths

If your workflow requires writing to locations outside the project directory:

```yaml
project:
  allowed_write_dirs:
    - .
    - /tmp/build-output
    - ~/shared-artifacts
```

### Allowing GPU access

By default the sandbox does not expose any GPU device nodes. ffmpeg and similar
tools fall back to slow CPU encode/decode paths. When your workflow involves
generating or processing video (or anything else that benefits from GPU
acceleration), enable GPU passthrough:

```yaml
sandbox:
  allow_gpu: true
```

This binds the minimal device surface needed for hardware acceleration to work
on every common vendor:

| Platform | Stack | What is bound |
|----------|-------|---------------|
| Linux | Intel QSV (`h264_qsv`, `hevc_qsv`, `av1_qsv`), Intel/AMD VA-API | `/dev/dri` (render + display nodes) |
| Linux | AMD AMF (`h264_amf`, `hevc_amf`), AMD ROCm | `/dev/dri`, `/dev/kfd` |
| Linux | NVIDIA NVENC / NVDEC (`h264_nvenc`, `hevc_nvenc`, `av1_nvenc`, `h264_nvdec`, ...) | `/dev/nvidia0`, `/dev/nvidiactl`, `/dev/nvidia-uvm`, `/dev/nvidia-uvm-tools` |
| Linux | All of the above | `--ro-bind /sys /sys` so the iHD / iris / radeonsi drivers can enumerate devices via `/sys/class/drm` |
| macOS | VideoToolbox (`h264_videotoolbox`, `hevc_videotoolbox`, `prores_videotoolbox`) | `(allow iokit-open)` |

Per-vendor device paths are probed for existence at sandbox-build time, so
enabling `allow_gpu` on a host that lacks a given vendor's stack is a no-op for
that vendor — it will not cause `bwrap` to fail. Driver libraries
(`libcuda.so`, `libnvcuvid.so`, `/usr/lib/.../dri/*`) are read from `/`, which
is already accessible read-only.

`allow_gpu` is off by default because GPU device nodes are DMA-capable and
represent a real expansion of sandbox blast radius. Opt in only when you need
hardware acceleration.

### Checking sandbox status

The TUI shows sandbox status in the session info. If the sandbox is disabled, a warning is displayed.

## Per-session toggle

The `/sandbox` slash command lets the user temporarily disable the sandbox
for a single session, without changing the server-level config. The full
behavior summary is:

- `/sandbox` — show the current sandbox status as an out-of-band message
- `/sandbox on` — enable sandbox for this session
- `/sandbox off` — disable sandbox for this session

The toggle is persisted in `.meta.json` (default: `sandbox_disabled = false`)
and survives server restart. The status message always shows both layers
(server-level and session-level) so the user can tell whether the toggle is
actually doing anything:

- When the server-level sandbox is disabled, both `/sandbox on` and
  `/sandbox off` write the per-session flag (so it takes effect if sandboxing
  is later enabled at the server level), but neither affects tool execution
  in the meantime.
- Subagents inherit the root session's effective `sandbox_disabled` state at
  execution time — so toggling `/sandbox off` on the parent immediately
  applies to its running children. The flag is not persisted in the child's
  `.meta.json`, and children cannot toggle their own.
