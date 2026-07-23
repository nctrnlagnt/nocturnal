# Agent Skills

## What

Skills are external capability packages that extend what an agent can do without
modifying the agent definition itself. A skill provides instructions, tools, and
context that get loaded into the agent's system prompt at session start.

## How it works

A skill is a directory containing a `SKILL.md` file. The `SKILL.md` describes
the skill's purpose and provides detailed instructions for the agent on how to
use it. When a skill is loaded, its contents are injected into the agent's
system prompt.

Skills are separate from agents for a reason: they let you mix and match
capabilities. A single agent can load multiple skills, and the same skill can
be used across different agents. This avoids duplicating instructions in every
agent definition that needs them.

### What skills look like

Each skill directory has a `SKILL.md` with a YAML frontmatter (name and
description) followed by the skill instructions:

```
my-skill/
  SKILL.md       # Description + instructions for the agent
  scripts/       # Helper scripts the skill's instructions reference
  references/    # Reference material
```

The agent reads `SKILL.md` and follows its instructions. Scripts and references
are invoked by the agent when needed — they are not loaded into the prompt
automatically.

### Available skills

Skills live in three locations:

- `~/.nocturnal/skills/` — instance-local skills (NOCTURNAL_HOME; includes shipped skills such as product docs)
- `~/.agents/skills/` — user-global skills
- `<project>/.agents/skills/` — project-local skills

Example skills include:

| Skill | Purpose |
|-------|---------|
| `code-search-refactor` | AST-aware code search and refactoring across multiple languages |
| `agent-tui` | Automate and test terminal UI applications |
| `agent-browser` | Automate browser interactions for web testing |
| `nocturnal-image-ansi` | Generate ANSI art from image files |
| `rust-skills` | Comprehensive Rust coding guidelines |
| `zig` | Idiomatic Zig development guidelines |

## Configuration

### Loading skills into an agent

Add skill names to the `skills` field in an agent's YAML frontmatter:

```yaml
---
name: code
description: Coding assistant
skills: ["code-search-refactor", "rust-skills"]
tools: [read, write, edit, bash]
---
```

Multiple skills can be listed. They are loaded in order, so later skills can
reference earlier ones.

### Disabling skills

Skills are auto-discovered from `~/.nocturnal/skills/`, `~/.agents/skills/`, and
the active project's `.agents/skills/`. The user can disable individual
skills globally via `skills.disabled` in `~/.nocturnal/config.yaml`:

```yaml
skills:
  disabled:
    - rust-skills
    - zig
```

A disabled skill is filtered out of the agent's system prompt and is not
loadable by name from any agent definition. Disabling does not delete the
skill from disk — re-enabling is a config edit and a `/reload` away.

The Settings UI exposes a **Skills** tab with one card per discovered skill
(name, description, on-disk path) and a single enabled toggle. Toggling a
card writes the per-skill entry under `skills/<name>` in the config tree,
which maps to `skills.disabled` in `config.yaml`.

See [Configuration Reference](configuration-reference.md#skills-configuration)
for the full config schema.

### Installing skills

Skills are directory-based. To install one, place it in `~/.agents/skills/`
(user-global) or `~/.nocturnal/skills/` (instance-local):

```
~/.agents/skills/
  agent-tui/
    SKILL.md
  code-search-refactor/
    SKILL.md

~/.nocturnal/skills/
  my-custom-skill/
    SKILL.md
```

### Creating a custom skill

1. Create a directory in `~/.nocturnal/skills/` or `~/.agents/skills/`
2. Add a `SKILL.md` with YAML frontmatter and instructions:

```markdown
---
name: my-skill
description: What this skill does and when to use it.
---

# My Skill

Instructions for the agent on how to use this skill...

## Commands

Description of available commands and workflows...
```

3. Reference it in an agent definition: `skills: ["my-skill"]`

The `description` in the frontmatter is important — it tells the agent (and
other agents that might delegate) when this skill is relevant.
