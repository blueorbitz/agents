# Goal Evaluation

Documentation for how the judge works. The authoritative judge prompt is in [JUDGE-PROMPT.md](JUDGE-PROMPT.md) — that file is what the judge receives. This file is for human readers.

## Verdicts

| Verdict    | Meaning                                                      |
| ---------- | ------------------------------------------------------------ |
| `continue` | The step moved the goal forward. Work remains. Loop again.  |
| `halt`     | The goal is achieved or cannot be achieved. Stop.            |
| `pivot`    | The current approach is wrong. The goal needs a new path.    |

## How the judge works

The agent makes a separate model call after each step. It reads `JUDGE-PROMPT.md` as the system prompt, then sends the goal state JSON and the agent's step summary as the user message. The judge returns structured JSON with a verdict, reasoning, evidence, and optional next step suggestion.

The judge does NOT see raw tool output — only the agent's summary of what the step did.

## Decision flow

The judge answers five questions in order. First decisive answer wins:

1. Is the goal achieved? → `halt`
2. Is the goal achievable? → `halt` if no
3. Did the step advance the goal? → `pivot` if no
4. Is the agent stuck (3+ steps, no progress)? → `pivot`
5. Otherwise → `continue`

## Evidence

Every verdict must cite concrete evidence. Vague reasoning is not evidence. If the judge cannot name evidence, the verdict is `pivot`.

## Fallback

If the agent cannot parse the judge's response as JSON, the verdict defaults to `pivot` and the agent asks the user what to do.

## Pivot handling

When the judge returns `pivot`, the agent proposes a new path and the user confirms. If no viable path remains, the goal is `halted`.

## Evaluation history

Every evaluation appends to `evaluation_history` in the goal state, creating an audit trail that survives across sessions.
