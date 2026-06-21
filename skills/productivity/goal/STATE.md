# Goal State Schema

Every goal lives as a JSON file in `goals/`. The filename is the goal ID.

## Schema

```json
{
  "id": "string — hyphenated slug, matches filename",
  "goal": "string — the user's current objective, may be revised",
  "goal_revisions": [
    {
      "goal": "string — the previous version of the objective",
      "at": "ISO-8601 timestamp",
      "reason": "string — why the goal was revised"
    }
  ],
  "created": "ISO-8601 timestamp",
  "updated": "ISO-8601 timestamp",
  "status": "active | halted | complete",
  "halt_code": "goal_achieved | unachievable | no_viable_path | user_halted | max_steps_reached | null",
  "halt_explanation": "string | null — human-readable reason when halted",
  "current_step": "string — description of what the agent is doing right now",
  "step_checkpoint": {
    "partial_result": "string | null — partial work done before interruption",
    "artifacts_touched": "string[] — files or resources modified during the interrupted step"
  },
  "completed_steps": [
    {
      "step_number": "number — sequential, starting at 1",
      "step": "string — what was done",
      "result": "string — outcome or artifact produced",
      "at": "ISO-8601 timestamp"
    }
  ],
  "evaluation_history": [
    {
      "at": "ISO-8601 timestamp",
      "verdict": "continue | halt | pivot",
      "reasoning": "string — why this verdict",
      "evidence": "string — what was observed to support the verdict",
      "agent_disagreement": "string | null — agent's objection if it believes the verdict is wrong"
    }
  ],
  "context": {
    "discovered_constraints": "string[] — constraints learned during execution",
    "open_questions": "string[] — unresolved questions blocking progress",
    "working_assumptions": "string[] — assumptions the agent is operating on"
  },
  "max_steps": "number — hard cap on total steps, default 20"
}
```

## Persistence rules

1. **Write on every mutation.** After creating, executing a step, or evaluating — write the file immediately. No batching.
2. **Read before every action.** Load the goal file at the start of each step to get the latest state. The agent never assumes state from memory.
3. **One file per goal.** Goals do not share state files. If a goal references another goal, store the reference as a string ID, not a nested object.
4. **Never delete active goals.** Halting sets `status: "halted"` and a `halt_code`. Deletion is a manual user act.
5. **Filename convention:** `goals/<id>.json` where `<id>` is a short hyphenated slug (e.g. `fix-auth-bug`, `add-dark-mode`).

## Goal lifecycle

```
created → active → (step → evaluate → step → evaluate → ...) → complete | halted
```

- **active**: the agent is working toward the goal
- **complete**: the goal is achieved; `halt_explanation` describes why
- **halted**: the goal is abandoned or paused; `halt_explanation` describes why
