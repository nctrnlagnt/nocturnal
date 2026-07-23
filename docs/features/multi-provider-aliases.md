# Multi-Provider Aliases

## What

Aliases give users stable names for model choices. An alias can point to one
provider/model target, a failover list of provider/model targets, or one other
alias using the existing `alias/<name>` target string.

This keeps agent definitions simple: agents can refer to purpose names such as
`edit`, `review`, or `memory-review`, while users change those names in one
place in config.

## Config

Every alias uses the same shape: an object with a `targets` list and an optional
`strategy`.

```yaml
aliases:
  fast:
    targets: [anthropic/claude-haiku-4-20250514]

  smart:
    targets:
      - anthropic/claude-sonnet-4-20250514
      - openai/gpt-4o
```

The desktop editor can represent this as one alias editor: name, strategy
dropdown, and target rows.

## Effort Levels

Aliases can optionally specify an effort level per target. Effort levels control
how much compute a model spends on reasoning (for example `low`, `medium`,
`high`, `xhigh`, or `max`). Effort is set exclusively via aliases; direct model
selection uses the provider default.

Use an object target row when a target needs effort:

```yaml
aliases:
  reasoning:
    targets:
      - target: anthropic/claude-sonnet-4-20250514
        effort: high
```

For failover aliases, each target can have its own effort level:

```yaml
aliases:
  smart:
    targets:
      - target: anthropic/claude-sonnet-4-20250514
        effort: max
      - target: openai/gpt-4o
        effort: high
```

Effort values are provider-native. What the user configures is what the API
receives:

- **OpenAI Responses API**: sent as `reasoning.effort` in the request body.
- **Anthropic API**: sent as `output_config.effort` in the request body.
- **Other providers**: not supported; effort is ignored.

## Selection Strategy

Multi-target aliases pick one target per session using a strategy. Single-target
aliases ignore the strategy because there is only one possible target.

- **`round_robin`** (default): target selection rotates globally across sessions
  to spread load. On quota failover, the next target is the cyclic successor of
  the failed target.
- **`fill_first`**: each session starts with the first target. On quota failover,
  the session falls through to later targets in linear order and never loops
  back to the first.

```yaml
aliases:
  # round_robin is implied
  mixed:
    targets:
      - anthropic/claude-sonnet-4-20250514
      - openai/gpt-4o

  # Explicit round_robin
  spread:
    strategy: round_robin
    targets:
      - openai/gpt-5
      - anthropic/claude-sonnet

  # Always start with the first, then fall through on quota
  cheap:
    strategy: fill_first
    targets:
      - openai/gpt-5-nano
      - anthropic/haiku
      - google/gemini-flash
```

Use `round_robin` when targets are roughly interchangeable. Use `fill_first`
when the first target is materially preferable and later targets are only
fallbacks.

## Alias References

A target string can be `alias/<name>` to make one alias refer to another alias.
This is for purpose mappings such as `edit`, `review`, or `memory-review`.

```yaml
aliases:
  cheap:
    strategy: fill_first
    targets:
      - openai/gpt-5-nano
      - anthropic/haiku

  edit:
    targets: [alias/cheap]

  review:
    targets: [alias/cheap]
```

`edit` and `review` resolve through `cheap`. They use `cheap`'s strategy,
targets, failover behavior, and per-target effort settings.

To keep the config easy to reason about, alias references are intentionally
restricted:

- An alias reference must be the only target in its alias.
- Alias references cannot set their own effort; configure effort on the
  referenced alias's concrete targets.
- Cycles such as `edit -> cheap -> edit` are config errors.

This means every target row uses the same GUI control: the user chooses either a
provider/model or an existing alias, stored as `alias/<name>`.

## Model Catalog

Every valid configured alias is a first-class entry in `ListAllModels` and the
`ModelsChanged` event. Its wire identity is `provider: "alias"`, `id: <name>`,
and `alias_targets` contains the resolved concrete targets in config order.

Provider discovery does not control alias membership. An alias remains visible
when a provider is unauthenticated, its model endpoint is unavailable, or a
configured model id is missing from the provider response. Provider models and
aliases may therefore be inspected and repaired independently; `/reload` is not
required to make configured aliases appear.

Alias rows do not copy pricing or context limits from one target because a
multi-target alias can execute against a different provider after failover.
Runtime displays use the concrete provider/model reported by session status.

## How it works

**Session start.** When a session uses `alias/smart`, the server resolves alias
references first, then selects one concrete provider/model target according to
the resolved alias's strategy. The session state keeps the user-facing alias form
(`provider="alias", model="smart"`) for display, while the selected provider/model
is stored separately in `alias_resolved`. Subsequent turns reuse the same
selected target to preserve provider-side cache behavior.

**Quota failover.** When the active target returns a quota-window error (for
example, `Usage limit reached for 5 hour`), the session switches to the next
failover target and retries immediately. It does not sleep through the window.

The failover list comes from the strategy:

- `round_robin`: remaining targets in cyclic order, excluding the selected one.
- `fill_first`: `targets[1..]` in linear order, so later targets are tried in
  sequence and the session never loops back to the first.

**All targets exhausted.** If every target in the list is rate-limited, the
session falls back to the normal wait-and-retry behavior for the last target
tried.

**Child sessions.** When a parent uses an alias, a child inherits the unresolved
alias reference rather than the resolved provider/model. The child gets its own
selection based on the alias's strategy, so a quota window on the parent's target
does not automatically propagate to the child.

**Server restart.** `alias_name` and `alias_failover_targets` are persisted to
`.meta.json`. On restart, the session re-resolves the alias using its configured
strategy and picks a fresh target. `alias_resolved` is not persisted because the
resolved provider/model may no longer be valid after config changes.

**/reload.** Reloading config updates alias definitions and provider instances.
Existing sessions keep their current resolved target until a failover or a new
session picks from the updated alias list.

**Transparent to the TUI.** The TUI sends `alias/smart` as before. The session
selector keeps showing the alias form so the user can see which alias is active.
The footer shows the resolved provider/model so the user knows which concrete
target is in use.

## Guidelines

- **Use the same model on each failover target.** Targets with different context
  windows or capabilities can cause confusing behavior. The feature does not
  validate this.
- **Use independent providers.** The point is independent quota windows. Two
  entries pointing at the same provider with the same API key do not help.
- **Two is usually enough.** Each additional target is a fallback. Two providers
  on independent quotas already gives near-continuous availability.
