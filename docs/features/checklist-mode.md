# Checklist Mode

## What

Checklist mode drives an agent toward a goal with a mandatory review gate before
completion. The agent cannot sign off on its work until a review agent has
examined it and approved. This ensures quality through enforced peer review.

## How it works

The workflow has six steps:

1. **Plan.** The agent creates a plan file describing the overall approach and a
   checklist file (markdown with `[ ]` items) breaking the work into discrete
   steps.

2. **Register.** The agent registers checklist mode, providing the paths to the
   plan and checklist files. From this point, the system tracks progress against
   the checklist.

3. **Work.** The agent works through each item, marking it `[x]` as it
   completes. The system monitors the checklist to track how much remains.

4. **Submit for review.** When all items are checked off, or when blocked on items the agent cannot complete (e.g., manual testing that requires user action), the agent submits the work for review. This signals that the agent believes the work is done or is as far as it can go.

5. **Review.** A review agent examines the completed work. It can either:
   - **Approve** — call `checklist_signoff` to close the checklist, allowing the working agent to complete the workflow.
   - **Request changes** — describe what needs to be fixed, sending the working agent back to step 3.

6. **Sign off.** The reviewer's signoff clears the parent's checklist state and terminates the review session. The parent reads the reviewer's approval message and gives its final report.

This cycle can repeat: if the review agent requests changes, the working agent
makes fixes, re-submits, and waits for approval again.

## Configuration

Checklist mode is available by default. No configuration is required to use it.

The review agent is determined by the server configuration. If you want to
customize which model or agent performs reviews, see the agent configuration in
`~/.nocturnal/config.yaml`.

## Usage

Checklist mode is invoked by the agent itself — you do not need to enable it
manually. When you give the agent a task that benefits from structured execution
with review, it will:

1. Create the plan and checklist files in your project directory
2. Register checklist mode
3. Work through items and submit for review

You can observe progress by reading the checklist file, which shows which items
are done (`[x]`) and which remain (`[ ]`).

Example checklist file:

```markdown
- [x] Add input validation to the API endpoint
- [x] Write unit tests for validation logic
- [ ] Update documentation with new parameters
- [ ] Add integration test for error responses
```

If the review agent requests changes, the checklist may gain new items or revert
completed ones. The agent will continue working until the review agent approves.
