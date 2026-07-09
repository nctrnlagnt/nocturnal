# Providers

## What

Nocturnal supports multiple LLM providers simultaneously. You can configure OpenAI-compatible, Anthropic-compatible, Google-compatible, and OpenAI Responses API-compatible providers — including ChatGPT OAuth via the `openai-responses-compatible` type. All providers are defined in `~/.nocturnal/config.yaml`. Each provider
declares an `api_key_env_var` slot — the name of the env var that holds
its key — and the actual key value lives in `~/.nocturnal/secrets/.env` or
the process environment. `config.yaml` never holds a literal key.

## How it works

Each provider entry in config defines a connection to an LLM API. The server builds provider instances at startup and re-reads them on `/reload`. Sessions reference providers by name when selecting a model.

Provider types determine how requests are formatted and which API protocol is used:

| Type | Protocol |
|------|----------|
| `openai-compatible` | OpenAI Chat Completions API |
| `anthropic-compatible` | Anthropic Messages API |
| `google` | Google Generative AI API |
| `openai-responses-compatible` | OpenAI Responses API (used for ChatGPT OAuth) |

## Configuration

Providers are defined under the `providers` key in `~/.nocturnal/config.yaml`:

```yaml
providers:
  anthropic:
    type: anthropic-compatible
    base_url: https://api.anthropic.com
    api_key_env_var: ANTHROPIC_API_KEY

  openai:
    type: openai-compatible
    base_url: https://api.openai.com/v1
    api_key_env_var: OPENAI_API_KEY

  openai-codex:
    type: openai-responses-compatible
    base_url: https://chatgpt.com/backend-api/codex
    api_key_env_var: OPENAI_API_KEY

defaults:
  provider: anthropic
  model: claude-sonnet-4-20250514
```

### Fields

| Field | Required | Description |
|-------|----------|-------------|
| `type` | Yes | One of `openai-compatible`, `anthropic-compatible`, `google`, `openai-responses-compatible` |
| `base_url` | Yes | API endpoint URL |
| `api_key_env_var` | Yes | Name of the environment variable that holds this provider's API key. The actual key value lives in `~/.nocturnal/secrets/.env` (or the process environment) — `config.yaml` only stores the variable name, never the key itself. |
| `default_model` | No | Model to use when no model is specified for this provider |
| `api_path` | No | API path appended to `base_url`. Defaults to `/v1` for OpenAI-compatible, empty for Anthropic-compatible |
| `models` | No | List of models with `id` and `name` fields, overriding the provider's default model list |

### Defaults

The `defaults` section sets which provider and model are used when a session doesn't specify one:

```yaml
defaults:
  provider: anthropic
  model: claude-sonnet-4-20250514
```

### Custom model list

If a provider offers models that aren't auto-discovered, or you want to rename them for clarity, use the `models` list:

```yaml
providers:
  my-provider:
    type: openai-compatible
    base_url: https://my-llm.example.com/v1
    api_key_env_var: MY_API_KEY
    models:
      - id: llama-3-70b
        name: Llama 3 70B
      - id: llama-3-8b
        name: Llama 3 8B
```

## Usage

### TUI commands

| Command | Description |
|---------|-------------|
| `/connect` | Add a new provider interactively (prompts for type, URL, key) |
| `/models` | List available models across all configured providers |
| `/reload` | Re-read `config.yaml` and `.env` without restarting the server |

### Switching providers at session start

When starting a session, specify the provider and model:

```
provider/model
```

For example, `openai/gpt-4o` or `anthropic/claude-sonnet-4-20250514`.

### Rotating keys

1. Edit `~/.nocturnal/secrets/.env` with the new key
2. Type `/reload` in the TUI
3. New requests use the updated key. In-flight requests continue on the old instance.

### Keyless providers

A provider entry whose `api_key_env_var` names a variable that is unset in
`~/.nocturnal/secrets/.env` and the process environment is still kept in
`config.yaml` and remains visible in the Settings UI. This lets you see what
is configured and fill in the missing key without having to re-add the
provider.

Keyless providers are filtered out at the use sites that need a working key
— model listings, `/models`, and session execution all skip them. Once the
key is added (and `/reload` runs), the provider becomes usable without
re-adding it.

## Anthropic-Specific Features

Anthropic-compatible providers support several advanced features unique to Claude models.

### Effort Levels

Anthropic models support configurable effort levels that control response quality and latency. Set the effort level on individual models:

```yaml
providers:
  anthropic:
    type: anthropic-compatible
    base_url: https://api.anthropic.com
    api_key_env_var: ANTHROPIC_API_KEY
    default_model: claude-sonnet-4-20250514
    models:
      - id: claude-sonnet-4-20250514
        name: Claude Sonnet 4
        effort: high  # low, medium, high, xhigh, or max
```

Available effort levels:

| Level | Description |
|-------|-------------|
| `low` | Fastest responses, lower quality |
| `medium` | Balanced speed and quality |
| `high` | Higher quality, moderate latency |
| `xhigh` | Maximum quality, higher latency |
| `max` | Best possible quality, highest latency |

### Bearer Token Authentication

Anthropic-compatible APIs (e.g., third-party Claude-compatible gateways) may require Bearer token authentication instead of Anthropic's default `x-api-key` header. Nocturnal automatically uses Bearer auth for all `anthropic-compatible` providers:

```yaml
providers:
  my-claude-gateway:
    type: anthropic-compatible
    base_url: https://my-claude-gateway.example.com
    api_key_env_var: GATEWAY_API_KEY
    # Bearer auth is automatically used for anthropic-compatible providers
```

## Advanced Provider Configuration

### Custom API Paths

Some providers include the API version in their base URL, requiring a custom or empty API path. Use `api_path` to control what gets appended to `base_url`:

```yaml
providers:
  # Provider with version already in base URL
  my-versioned-provider:
    type: openai-compatible
    base_url: https://api.example.com/api/coding/paas/v4
    api_path: ""  # Empty - base URL already has version
    api_key_env_var: MY_API_KEY

  # Standard OpenAI-compatible (defaults to /v1)
  openai:
    type: openai-compatible
    base_url: https://api.openai.com
    # api_path defaults to /v1 if not specified
    api_key_env_var: OPENAI_API_KEY

  # Custom API path
  custom-provider:
    type: openai-compatible
    base_url: https://api.example.com
    api_path: /api/v2
    api_key_env_var: CUSTOM_API_KEY
```

Resulting endpoints:

| Provider | Chat Endpoint |
|----------|---------------|
| `my-versioned-provider` | `https://api.example.com/api/coding/paas/v4/chat/completions` |
| `openai` | `https://api.openai.com/v1/chat/completions` |
| `custom-provider` | `https://api.example.com/api/v2/chat/completions` |

### Provider-Side Tools

Some providers offer built-in tools that execute on the provider side (e.g., web search, code execution). These tools are passed through to the API without requiring local implementation.

Configure provider-side tools in agent definitions:

```yaml
# ~/.nocturnal/agents/researcher.md
---
name: researcher
model: claude-sonnet-4-20250514
provider_side_tools:
  - web_search_preview
  - code_execution
---

You are a research assistant with web search capabilities.
```

Provider-side tools:

- Execute on the provider's infrastructure
- Don't require local tool definitions
- Are passed as-is to the API request
- Useful for capabilities like real-time web search

Common provider-side tools:

| Tool | Providers | Description |
|------|-----------|-------------|
| `web_search_preview` | xAI, OpenAI | Real-time web search |
| `code_execution` | OpenAI | Python code execution |

### Model Configuration Fields

Individual models can be configured with additional fields:

```yaml
providers:
  custom:
    type: openai-compatible
    base_url: https://api.example.com
    api_key_env_var: API_KEY
    models:
      - id: model-x
        name: Model X
        description: High-performance model
        input_price: 1.5      # Cost per million input tokens
        output_price: 3.0    # Cost per million output tokens
        context_limit: 100000  # Maximum context length
        output_limit: 8000     # Maximum output tokens
        effort: high           # Anthropic effort level (anthropic-compatible only)
```

| Field | Description |
|-------|-------------|
| `id` | Model identifier used in API requests |
| `name` | Human-readable display name |
| `description` | Model description for UI |
| `input_price` | Cost per million input tokens |
| `output_price` | Cost per million output tokens |
| `context_limit` | Maximum context window size |
| `output_limit` | Maximum response length |
| `effort` | Anthropic effort level (anthropic-compatible only) |
