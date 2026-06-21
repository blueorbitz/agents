# Glossary

Bold terms from the goal skill, defined here.

**goal** — A persistent objective the agent works toward. Defined by the user, refinable with revision history. Each goal has a unique ID and a JSON state file in `goals/`.

**step** — A logical unit of work toward a goal. Big enough to produce a clear artifact, small enough that mistakes are cheap to fix. Steps are sequential, numbered, and recorded in `completed_steps`.

**halt** — The terminal state where the goal is either achieved (`complete`) or abandoned (`halted`). Once halted, the loop stops. The judge returns the `halt_code` that classifies why.

**pivot** — A decision point where the current approach has failed. The agent proposes a new path, the user confirms, and work continues under a different strategy.

**judge** — A separate model call that evaluates the goal state after each step. Receives the goal state and step summary, returns a structured JSON verdict. Answers five questions in order; first decisive answer wins.

**verdict** — The output of a judge evaluation: `continue` (keep going), `halt` (stop), or `pivot` (change approach).

**goal state** — The JSON file `goals/<id>.json` containing the goal's ID, objective, status, steps, evaluation history, and context. The single source of truth.

**active** — Goal status meaning the agent is currently working toward the objective.

**complete** — Goal status meaning the objective is achieved.

**halted** — Goal status meaning the goal is abandoned or paused, with a `halt_code` and `halt_explanation`.

**halt_code** — An enum that classifies why a goal halted: `goal_achieved`, `unachievable`, `no_viable_path`, `user_halted`, `max_steps_reached`.

**persistent state** — Goal data written to disk on every mutation and read before every action. The agent never assumes state from memory.

**context** — Structured data the agent carries between steps: `discovered_constraints`, `open_questions`, `working_assumptions`. Stored in the goal state file, not in session memory.

**slug** — A hyphenated identifier derived from the goal description (e.g. `fix-auth-timeout`). Used as the filename for the goal state file.

**evidence** — Concrete, nameable proof cited by a judge verdict. Vague reasoning is not evidence. If you cannot name the evidence, the verdict is `pivot`.

**resumption** — The act of loading an existing goal state and continuing the loop from where it left off.

**checkpoint** — Partial work saved when a step is interrupted. Allows the agent to resume from the interruption point instead of restarting the step.
