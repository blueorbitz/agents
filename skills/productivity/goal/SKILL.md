---
name: goal
description: Hermes-style goal loop with persistent state. Triggers: /goal, "set a goal", "work toward".
disable-model-invocation: true
---

# Goal Loop

A goal is a persistent, stateful objective the agent works toward across steps. Each run: execute one step, evaluate the result, decide — loop or halt. State survives sessions.

## Definitions

- **goal**: the objective, stated by the user, refinable with revision history
- **step**: a logical unit of work that produces a clear artifact the judge can evaluate (e.g. implementing a function, fixing a bug, writing a test — not a single tool call)
- **halt**: the goal is done or cannot proceed
- **pivot**: the current approach failed; the goal needs a new path
- **judge**: a separate model call that evaluates the goal state after each step and returns a verdict

Full definitions in [GLOSSARY.md](GLOSSARY.md).

## Creating a goal

1. Capture the user's objective verbatim in `goal`.
2. Generate a hyphenated slug for `id` (e.g. `fix-auth-timeout`).
3. Write the initial state to `goals/<id>.json` — see [STATE.md](STATE.md) for the schema.
4. Propose the first step and a rough plan to the user. Wait for confirmation before proceeding.
5. Set `status: "active"`, `current_step` to the confirmed first action, `completed_steps: []`, `evaluation_history: []`, `max_steps: 20` (or user-specified), `goal_revisions: []`.
6. Confirm the goal and its first step to the user.

Completion criterion: goal file exists on disk with `status: "active"` and a non-empty `current_step`.

## Revising a goal

A goal can be revised mid-loop. Two triggers, same mechanic:

1. **User-initiated** — the user says something like "actually, the goal should be X" or "change the objective to Y". The agent detects the revision intent, confirms with the user, then updates.
2. **Agent-initiated** — during a pivot, the agent can propose a goal revision (not just a path change). The agent says "your goal may be wrong — here's a better one: X". The user confirms or rejects.

In both cases, before applying a revision:
- Show the current goal and the proposed new goal side by side
- Wait for explicit user confirmation
- If confirmed: append the old goal to `goal_revisions` with timestamp and reason, update `goal` to the new text
- If rejected: keep the current goal, continue the loop

## Executing a step

1. Load the goal file from `goals/<id>.json`. Never assume state from memory.
2. Execute the action described in `current_step`. A step is a logical unit of work — big enough to produce a clear artifact, small enough that mistakes are cheap to fix. Produce a concrete artifact or observable change — a file written, a command run, a decision recorded.
3. If a tool call fails, retry up to 3 times for transient errors (network, file locks). If it still fails, record the failure in `completed_steps` with the error message and let the judge decide.
4. Update the goal file: append to `completed_steps` with `step_number` (one more than the last entry), the step description, result, and timestamp. Set `current_step` to the next logical action.

Completion criterion: `completed_steps` has one more entry than before the step, and `current_step` is non-empty.

## Evaluating

After each step, make a separate model call to judge the goal state. The agent reads [JUDGE-PROMPT.md](JUDGE-PROMPT.md) and sends it as the system prompt, with the goal state and step summary as the user message. The judge returns a structured JSON verdict. The judge answers in order — first decisive answer wins:

1. Is the goal achieved? → `halt`
2. Is the goal still achievable? → `halt` if no
3. Did the step advance the goal? → `pivot` if no
4. Is the agent stuck? (3+ steps with no progress) → `pivot`
5. Otherwise → `continue`

Full evaluation rules in [EVALUATION.md](EVALUATION.md).

1. Append the verdict to `evaluation_history` with reasoning and evidence.
2. If `continue`: print a structured status block, then loop back to executing a step:
   ```
   Step N: <what was done>
   Judge: continue — <evidence>
   Next: <next step>
   ```
3. If `pivot`: print the pivot block, acknowledge the failed approach, propose a new path, confirm with the user, then loop:
   ```
   Step N: <what was done>
   Judge: pivot — <evidence>
   Proposed: <new approach>
   ```
4. If `halt`: print the halt block, set `status: "complete"` or `"halted"` with `halt_code` and `halt_explanation`. Save and stop:
   ```
   Step N: <what was done>
   Judge: halt — <evidence>
   Goal: <complete | halted> (<halt_code>)
   ```

The loop runs autonomously — the agent does not wait for user permission between steps. The user can interrupt at any time. The brief status line after each step lets the user see progress and intervene.

Completion criterion: goal file has `status: "complete"` or `"halted"`, or a new step is about to execute.

## Resuming a goal

To resume a paused or halted goal:

1. Load `goals/<id>.json`. If the file is missing or corrupted, report the error and show the user what a valid goal file looks like. Do not proceed until the file is fixed.
2. Validate the file has required fields: `id`, `goal`, `status`, `current_step`, `completed_steps`, `evaluation_history`, `max_steps`. If any are missing, report which ones and show the expected schema.
3. If `status: "halted"` and the user wants to continue, set `status: "active"` and propose a new `current_step`.
4. If `status: "complete"`, tell the user the goal is already done.
5. If `step_checkpoint.partial_result` is set, inform the user that the previous step was interrupted and offer to resume from the checkpoint or restart the step.
6. Display a structured resume block:
   ```
   Goal: <goal> (step N/max, <status>)
   Last: <last step result> — judge: <last verdict>
   Next: <current_step>
   ```
7. Continue the loop from the current state.

## Listing goals

Read all `*.json` files in `goals/`. Display each as: `id` | `status` | first 60 chars of `goal`.

## Deleting a goal

Soft delete only. The agent moves `goals/<id>.json` to `goals/.archived/<id>.json`. The goal disappears from listings but is recoverable. Permanent deletion is a manual act by the user on the `.archived/` directory.

## Concurrent goals

Any number of goals can have `status: "active"`. If the user names a goal, work on that one. If no goal is named, continue the most recently updated active goal. Each goal's state is fully isolated in its own file — no shared state between goals.
