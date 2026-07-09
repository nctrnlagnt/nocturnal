# Context Compaction

## What

Automatic context compaction allows agent workflows to run indefinitely, even on
tasks that generate far more conversation than a single context window can hold.
When a conversation grows too long, the system automatically summarizes earlier
content and keeps recent turns intact. This frees up space for new content
without losing important context.

## How it works

LLMs have a fixed context window — the total number of tokens (input + output)
they can process in a single request. Long-running tasks (large refactors,
multi-file changes, extended debugging sessions) can exceed this limit.

When the conversation approaches the context window limit, the system:

1. **Summarizes earlier turns.** A summary of the conversation so far is
   generated, preserving key decisions, facts, and the current state of work.
2. **Keeps recent turns.** The most recent messages remain intact so the agent
   has full context for its current task.
3. **Replaces old content with the summary.** The summarized portion replaces
   the original messages, freeing token capacity.

The agent continues working as normal — it does not need to restart or lose
track of what it was doing. Compaction is transparent to the workflow.

Compaction also triggers a refresh of cross-session memory, so any new
preferences or project facts discovered during the conversation are captured
before the detailed context is summarized away.

When the session is being compacted, the agent selection in the TUI remains unchanged.
The user's agent selection represents their intent and must never be overridden by the server.

**Agent Selection Principle:**

The TUI has two distinct agent displays:

1. **Assistant response footer** - Shows the agent name for the current streaming message
2. **Agent selection in input area** - The user's choice of agent for their next message

**Critical rule:** The agent selection in the input area represents the user's intent and must **never**
be changed by the server during a session. The user is in full control from the point of loading onward.

- When loading a session, the agent selection reflects the agent used for the last response
- During a session, the agent selection **never** changes from what the user has selected
- The server must respect user agency at all times

The assistant response footer may show different information during special operations (like compaction),
but this must not affect the user's agent selection.

## Configuration

Compaction happens automatically based on context window size. No configuration
is required.

The compaction threshold is determined by the model's context window and cannot
be manually overridden. This is intentional — compaction at the right moment
prevents failed requests from exceeding the context limit.

## Usage

Compaction runs automatically. You do not need to do anything.

You can also trigger compaction manually with the `/compact` command in the TUI:

```
/compact              # Compact now with default behavior
/compact <argument>   # Compact with optional argument
```

Manual compaction is useful when you want to free up context before starting a
new phase of work, or when you notice the agent is approaching the context
limit and want to compact on your terms rather than waiting for the automatic
trigger.
