# Persistent Python REPL (RLM)

## What

The Recursive Language Model (RLM) skill provides a persistent Python REPL
environment that preserves state between calls. Variables, imports, and
function definitions carry over from one execution to the next within a session.

## How it works

The REPL runs as a background server process. Each call sends code to the
server, which executes it in the same Python process. Output is returned
without the full context needing to be resent.

This avoids **context rot** — the degradation that happens when large outputs
fill up the conversation history. Instead of pasting entire datasets or
intermediate results back and forth, the REPL holds them in memory. Only the
final result needs to appear in the conversation.

## Usage

The RLM skill is invoked by the agent. You can ask the agent to run Python
code, and it will use the persistent REPL automatically.

Common patterns:

- **Incremental analysis.** Load a dataset once, then run multiple queries
  against it without reloading.
- **Stateful computation.** Define functions and classes in one call, use them
  in the next.
- **Large data processing.** Keep large objects in the REPL's memory rather
  than round-tripping them through the conversation.

Examples of what you might ask an agent to do:

> "Load the CSV at data/metrics.csv and show me the first 5 rows."

> "Now filter to just the last 7 days and plot the trend."

> "Save that plot to output/trend.png."

Each step builds on the previous one without re-loading the data.
