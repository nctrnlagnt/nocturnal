# Dot-Env File (~/.nocturnal/secrets/.env)

## What

A private file for storing API keys and other secrets that nocturnal uses
to resolve provider `api_key_env_var` slots at startup and on `/reload`.
Variables in `.env` are **not** injected into the process environment, so
child processes (bash, tools, shells) cannot access them.

## Why

API keys in `~/.bashrc` or `~/.zshrc` are visible to every process in your
shell session. Any tool or script can read them via `/usr/bin/env` or
`printenv`. The `.env` file keeps keys private to nocturnal's config expansion
only.

## Setup

Create `~/.nocturnal/secrets/.env` with one `KEY=VALUE` per line:

```
# API keys for nocturnal
ANTHROPIC_API_KEY=sk-ant-...
OPENAI_API_KEY=sk-...
GOOGLE_API_KEY=AIza...
```

Then declare the variable names in `~/.nocturnal/config.yaml` using the
`api_key_env_var` field on each provider. The provider definition only
names the variable; the actual key value never lives in `config.yaml`.

```yaml
providers:
  anthropic:
    type: anthropic-compatible
    api_key_env_var: ANTHROPIC_API_KEY

  openai:
    type: openai-compatible
    api_key_env_var: OPENAI_API_KEY
```

## Resolution order

When nocturnal resolves a provider's `api_key_env_var` slot, it checks two
sources in priority order:

1. **`~/.nocturnal/secrets/.env`** — highest priority. If a variable is defined here, this
   value wins.
2. **Process environment** (`~/.bashrc`, `~/.zshrc`, etc.) — fallback. If a
   variable is not in `.env`, the shell environment is checked.

This means `.env` can **override** shell values without modifying your
`.bashrc`. And if you prefer keeping keys in your shell environment, you can
skip `.env` entirely — the same `config.yaml` works either way.

## Format

```
# Comments start with #
BLANK_LINES_ARE_IGNORED=yes

# Optional "export" prefix (stripped automatically)
export SOMETIMES_PEOPLE_WRITE=this

# Values can be quoted
DOUBLE_QUOTED="allows spaces"
SINGLE_QUOTED='also works'

# Last value wins for duplicate keys
SAME_KEY=first
SAME_KEY=second    # this one is used
```

## /reload command

The `/reload` slash command re-reads both `~/.nocturnal/secrets/.env` and `config.yaml`
without restarting the server. This lets you rotate API keys or change
provider configuration on the fly:

1. Edit `~/.nocturnal/secrets/.env` (e.g., paste a new API key)
2. Type `/reload` in the TUI
3. nocturnal re-reads `.env`, re-resolves every provider's `api_key_env_var`
   slot, and rebuilds provider instances with the updated values

Any in-flight requests continue on the old provider instance. New requests
use the updated configuration.

## Security properties

- `.env` values are **never** injected into the process environment
- Child processes spawned by tools (bash, scripts, etc.) cannot see `.env`
  keys via `env`, `printenv`, or `/proc/$PID/environ`
- The `.env` file should be readable only by your user: `chmod 600 ~/.nocturnal/secrets/.env`
- The sandbox's `hide_nocturnal_home` setting (default: true) also prevents
  sandboxed agents from reading `~/.nocturnal/secrets/.env` directly
