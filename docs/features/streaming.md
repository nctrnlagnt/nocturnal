# Streaming

## What

Nocturnal streams LLM responses in real-time, including both reasoning ("thinking") and text content. You see output as it arrives — no waiting for the full response to complete before anything appears.

## How it works

**Chunk-level streaming.** The server forwards tokens from the provider as they arrive. Reasoning content and text content are streamed independently, so you see the model's thought process and its final output in parallel.

**Cache-preserving round-robin.** When multiple provider targets are configured (via multi-provider aliases), sessions stick with one target for the duration of a turn. This preserves the provider's prompt prefix cache — switching targets mid-conversation would invalidate the cache and increase both latency and cost. Round-robin selection happens at session start, not mid-turn.

**Failover without interruption.** If the active target hits a quota limit, the session switches to the next target and retries immediately. Long workflows continue without manual intervention.

**Zero re-render in the TUI.** The TUI renders each new chunk incrementally — it appends to the existing output without re-parsing or re-rendering the entire message. A full re-parse and re-render only happens when the terminal width changes (e.g., resizing the window). This keeps the UI responsive even during fast streaming or when many messages are on screen.

## Configuration

Streaming behavior is automatic and requires no configuration. It works with all supported provider types.

To configure multiple targets for failover, see [Multi-Provider Aliases](multi-provider-aliases.md).

## Usage

No action is needed — streaming is always active. In the TUI:

- **Reasoning** appears in a dimmed style as the model thinks
- **Text content** renders with full formatting as it arrives
- **Tool calls** appear as they are issued, with results streaming back in real-time

If a provider target becomes unavailable, the session automatically fails over to the next configured target. The TUI footer shows the currently active provider/model so you can see when a switch occurs.
