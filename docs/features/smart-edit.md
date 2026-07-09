# Smart Edit

## What

When using a smart or expensive model, the edit agent delegates file editing to a cheaper, faster model. The parent agent plans the edits; the edit agent does the mechanical work of reading files and applying changes. This can effectively double your quota on request-limited plans.

## How it works

**Delegation, not duplication.** The parent agent decides what needs to change and describes the edits. The edit agent receives those instructions, reads the relevant files, and applies the changes using a cheaper model. The expensive model never wastes tokens on reading file contents or formatting edits — it only does the thinking.

**Token-efficient edits.** The edit tool minimizes unnecessary token usage by sending only the information needed to make each change, rather than re-sending entire file contents.

**Separate agent definition.** The edit agent is configured as its own agent entry with a different model or alias. This means you can pair any smart model with any cheap model — the pairing is fully configurable.

## Configuration

The edit agent is defined in `~/.nocturnal/agents/edit.md` as a markdown file with YAML frontmatter, just like any other agent. The key is to set its `model` to a cheaper alias or provider/model pair:

```markdown
---
name: edit
description: Lightweight editing agent
model: fast
provider: alias
tools: [read, write, edit, bash]
agents: []
user_selectable: false
---

You are an expert source code editor...
```

The `provider: alias` with `model: fast` resolves to whatever your `fast` alias points to (configured in `~/.nocturnal/config.yaml`). See [Agent Definitions](agent-definitions.md) for the full frontmatter format.

The parent agent references the edit agent by name. When the parent decides to edit a file, it delegates to the `edit` agent automatically.

## Usage

No manual action is required. When the parent agent needs to edit a file, it delegates to the edit agent transparently. You see the edits appear in the session as they are applied.

### Choosing the edit model

The edit model needs to be competent at following precise instructions, but it doesn't need to plan or reason. Good choices:

- Fast, cheap models from any provider
- Models on independent quota windows (so edit operations don't consume the parent's quota)
- Models with low latency (since edits are interactive)

### When smart edit is used

Smart edit is triggered when the parent agent uses the edit tool. If no edit agent is configured, the parent agent performs edits directly using its own model — which works, but consumes more of your expensive quota.
