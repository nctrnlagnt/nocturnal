# Cron — Scheduled and Recurring Tasks

## What

Cron lets agents schedule jobs that run automatically on a recurring schedule or
as one-shot delayed tasks. Each job spawns a fresh ephemeral agent session when
it fires. Jobs can run ad-hoc prompts, execute scripts, or both.

## How it works

**Jobs are managed via the `cron` tool.** Agents use the tool with an `action`
parameter to create, edit, list, pause, resume, or remove jobs. The available
actions are:

| Action | Description |
|--------|-------------|
| `help` | Show all actions and input formats |
| `list` | Show all jobs in a table |
| `show` | Full detail for one job (requires `job_id`) |
| `create` | Create a new job |
| `edit` | Update fields on an existing job |
| `remove` | Delete a job |
| `pause` | Stop a job from running (keeps it defined) |
| `resume` | Re-enable a paused job |
| `run` | Trigger a job immediately |

**Jobs inherit the creating session's security context.** When a job fires, the
spawned session uses the same working directory, agent, and security settings as
the session that created it.

**Only named (non-UUID) sessions can create cron jobs.** The creating session
must exist at fire time to provide the sandbox profile and provider/model for
inheritance. Named sessions are persistent; UUID sessions may be evicted or
completed before the job fires.

**Jobs take effect within 60 seconds.** The scheduler checks for due jobs once
per minute. Editing a job's schedule resets its next-run time.

**Cron-spawned sessions inherit the creating session's security context.**
The spawned session uses the same working directory, agent, and security
settings as the session that created it, via the standard
`sessions_spawn` parent inheritance path. A synthetic "cron" session
acts as the sender for the spawn call (for tool-visibility and event
isolation), but the creating session is the actual parent for
inheritance. The cron session is ephemeral and hidden from session
lists.

**Job output is archived automatically.** Each run's result is saved to disk
under `~/.nocturnal/plugins/cron/output/<job_id>/`. Jobs can also deliver results to
Telegram or Discord via the `deliver` field.

## Configuration

Jobs are stored in `~/.nocturnal/plugins/cron/jobs.json`. You can edit this file
directly or manage jobs entirely through the `cron` tool.

### Schedule formats

| Format | Example | Behavior |
|--------|---------|----------|
| Duration | `30m`, `2h`, `1d` | One-shot, runs once after the delay |
| Interval | `every 30m`, `every 2h` | Recurring at the given interval |
| Cron expression | `50 8 * * *` | Recurring, standard 5-field cron (local timezone) |
| ISO timestamp | `2026-06-01T09:00` | One-shot at an absolute time |

### Create/edit fields

| Field | Required | Description |
|-------|----------|-------------|
| `schedule` | Yes (create) | When and how often the job runs |
| `prompt` | Yes (create) | Self-contained instruction for the agent |
| `name` | No | Friendly display name |
| `deliver` | No | How to send results. Default `"local"` (archive only). Use `"telegram:<chat_id>"` or `"discord:#channel"` to send results externally |
| `repeat` | No | Integer limit on total runs. Omit = forever for recurring, or set `1` for one-shot |
| `script` | No | Relative path to a script under `<project>/.nocturnal/scripts/` or `~/.nocturnal/scripts/`. Script stdout is injected as context |
| `no_agent` | No | Set `true` to run the script only, without invoking an LLM (requires `script`) |
| `model` | No | Override provider and model, e.g. `{"provider":"openrouter","model":"claude-sonnet-4"}` |
| `time_limit_secs` | No | Maximum runtime for the spawned session |

## Usage

Agents create and manage jobs through the `cron` tool. You can ask an agent to
set up scheduled tasks in natural language, or invoke the tool directly through
the agent.

Examples:

Create a recurring job that checks disk space every 2 hours:

```
cron(action="create", input='{"schedule":"every 2h","prompt":"Check disk space and report any volumes above 80% usage"}')
```

Create a one-shot job to run at a specific time:

```
cron(action="create", input='{"schedule":"2026-06-01T09:00","prompt":"Generate the weekly report","name":"weekly-report"}')
```

List all jobs:

```
cron(action="list")
```

Pause and resume a job:

```
cron(action="pause", input='{"job_id":"nightly-build"}')
cron(action="resume", input='{"job_id":"nightly-build"}')
```

Trigger a job immediately:

```
cron(action="run", input='{"job_id":"nightly-build"}')
```
