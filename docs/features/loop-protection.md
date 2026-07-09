# Loop Protection

## What

Automatic mitigations for common LLM error patterns. When an agent gets stuck in
a loop, receives malformed tool results, or overthinks a problem, the system
detects and corrects the situation without manual intervention.

## How it works

**Looping behavior circuit breaker.** LLMs sometimes get stuck repeating the
same actions — calling the same tool with the same arguments, or cycling through
a fixed set of steps without making progress. The circuit breaker detects this
pattern and breaks the loop by injecting a corrective message that tells the
agent to try a different approach.

**Malformed tool results.** Tools can return unexpected output — missing fields,
wrong types, or garbled text. Instead of crashing or passing the broken result
to the agent (which often causes cascading errors), the system intercepts
malformed results and presents a clean error to the agent so it can react
appropriately.

**Excessive reasoning mitigation.** Some models overthink simple problems,
producing very long reasoning chains that consume tokens and time without
improving the result. The system detects when reasoning exceeds a productive
threshold and nudges the agent toward action.

All three protections run automatically. They do not require configuration and
cannot be disabled.

## Configuration

No configuration. Loop protection is always active.

## Usage

Loop protection is invisible — you do not need to enable or configure it. When
a protection triggers, you may see the agent change approach mid-workflow (e.g.,
switching from a failing strategy to an alternative). This is the circuit
breaker or mitigation doing its job.

If you notice an agent repeatedly triggering loop protection on a particular
task, consider whether the task itself is ambiguous or underspecified — the
agent may need clearer instructions rather than more retries.
