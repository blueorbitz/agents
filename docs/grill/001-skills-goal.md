# Grill Session 001: Goal Skill Design

Date: 2026-06-22
Skill: `skills/engineering/goal/`
Method: Relentless one-at-a-time interview, resolving dependencies between decisions.

## Summary Table

| # | Decision | Recommendation | Choice |
|---|----------|---------------|--------|
| 1 | Judge identity | Separate model call | 2 — Separate model call |
| 2 | Autonomy level | Hybrid | 3 — Hybrid |
| 3 | Step granularity | Logical unit of work | 2 — Logical unit of work |
| 4 | Context field | Drop it entirely | 2 — Structured key-value |
| 5 | Goal mutability | Refinable with history | 2 — Refinable with history |
| 6 | Concurrent goals | Multiple active, interleaved | 2 — Multiple active, interleaved |
| 7 | Error handling | Retry 3x + judge decides | Follow recommendation |
| 8 | halt_reason structure | Two fields replacing one | 3 — Two fields replacing one |
| 9 | Max steps | Configurable, default 20 | 2 — Configurable, default 20 |
| 10 | Goal selection | Default to recency, allow override | 4 — Default to recency, allow override |
| 11 | Judge input | Goal state + last step raw output | 3 — Goal state + agent summary |
| 12 | Judge output format | JSON with fallback | 3 — JSON with fallback |
| 13 | Judge prompt location | Separate prompt file | 2 — Separate prompt file |
| 14 | Goal revision mechanism | Natural language detection | 2 and 3 — Natural language + agent-initiated |
| 15 | Step numbering | Array index only | 2 — Explicit step_number field |
| 16 | Status line format | Structured block (3 lines) | 2 — Structured block |
| 17 | Halt code mapping | Judge with a lookup | 1 — Judge returns it directly |
| 18 | First step creation | Collaborative | 3 — Collaborative |
| 19 | Invocation type | Model-invoked | 2 — User-invoked |
| 20 | Goal deletion | Soft delete | 3 — Soft delete |
| 21 | Agent disagrees with judge | Agent can flag but not override | 3 — Agent can flag but not override |
| 22 | User interruption | Step is lost | 3 — Agent checkpoints |
| 23 | Goal file unreadable | Fail loudly with guidance | 3 — Fail loudly with guidance |
| 24 | Resume display | Structured resume block (3 lines) | 3 — Structured resume block |
| 25 | Judge source of truth | JUDGE-PROMPT.md | 1 — JUDGE-PROMPT.md |

## Detailed Questions

### Q1: Who is the judge?

**Context:** The skill says "judge evaluates the goal state" but doesn't specify who the judge is. This determines whether the loop is self-correcting or self-reinforcing.

**Options:**
1. The same agent — executes and evaluates. Zero extra cost, but self-grading bias.
2. A separate model call — different model or fresh system prompt. Breaks bias, adds one LLM call per step.
3. The user — after each step, agent presents result and asks user to judge. Maximum accountability, kills autonomous loop.

**Recommendation:** Option 2 — a separate model call. Breaks self-grading bias while keeping the loop autonomous. Cost is one extra LLM call per step, cheap compared to the step itself.

**Choice:** 2 — Separate model call.

---

### Q2: How does the loop run?

**Context:** The loop promises autonomous multi-step progress, but "autonomous" has a spectrum.

**Options:**
1. Fully autonomous — agent loops without stopping. User set the goal and walks away.
2. Step-gated — agent executes one step, shows result, waits for user "continue" before looping.
3. Hybrid — autonomous by default, user can intervene. Agent prints brief status after each step, doesn't block on user input.

**Recommendation:** Option 3 — hybrid. The whole point is predictability through autonomy, but a fully unattended loop that goes wrong wastes resources silently. A brief status line costs almost nothing and gives the user a window to pull the plug.

**Choice:** 3 — Hybrid.

---

### Q3: What is a step?

**Context:** The skill says "step" is a unit of execution, but never defines how big a step is. This determines judge overhead and error-correction speed.

**Options:**
1. One tool call — writing a file, running a command. Very fine-grained. Most steps too small to evaluate meaningfully.
2. One logical unit of work — implementing a function, fixing a specific bug, writing a test. Big enough for clear pass/fail, small enough mistakes are cheap to fix.
3. One user-visible milestone — a complete feature, a working integration. Judge fires rarely, mistakes compound longer.

**Recommendation:** Option 2 — one logical unit of work. A step should be the smallest thing that produces a clear artifact the judge can evaluate. The agent decides step boundaries; the judge only fires when there's something worth judging.

**Choice:** 2 — Logical unit of work.

---

### Q4: What goes in `context`?

**Context:** The state schema has a `context` field: `"string | null — any extra context the agent carries between steps"`. It's the vaguest field in the schema.

**Options:**
1. Free-form string — agent dumps whatever it thinks is important. Flexible, but no structure means no guarantees.
2. Structured key-value — object with defined keys like `discovered_constraints`, `open_questions`, `working_assumptions`. Readable, but rigid.
3. No `context` field at all — everything the agent needs is in `completed_steps` and `evaluation_history`. If the agent can't reconstruct context from step history, the step was too small.

**Recommendation:** Option 3 — drop `context` entirely. The audit trail IS the context. A separate `context` field is a redundancy that drifts out of sync.

**Choice:** 2 — Structured key-value. (User wanted structured context despite the redundancy argument.)

---

### Q5: Can the user change the goal mid-loop?

**Context:** The skill says the goal is "immutable once set." But users change their minds, discover new constraints, or realize the original goal was wrong.

**Options:**
1. Immutable — once set, goal string never changes. User halts and creates a new one. Clean audit trail, but rigid.
2. Refinable with history — user can update the `goal` field, old version appended to `goal_revisions`. Judge evaluates against latest version. Flexibility plus traceability.
3. Mutable, no history — user can change goal at any time. Simple, but evaluation history now evaluates against a moving target. Audit trail broken.

**Recommendation:** Option 2 — refinable with history. The judge needs to evaluate against the current goal, and the user needs flexibility. Recording revisions keeps the audit trail intact.

**Choice:** 2 — Refinable with history.

---

### Q6: Can multiple goals be active at once?

**Context:** The current design assumes one goal at a time. But users often juggle multiple tasks.

**Options:**
1. One active goal — only one goal can have `status: "active"`. Simple, forces focus, but annoying if user wants to switch context.
2. Multiple active goals, interleaved — any number can be active. Agent picks one per invocation. Each goal has its own state file. More flexible, agent might lose focus.
3. Multiple active goals, one at a time — multiple active, but agent only works on one per session. User specifies which. This is option 1 with a queue.

**Recommendation:** Option 2 — multiple active goals, interleaved. State files are already isolated by ID. User says which goal to work on. Only cost is the agent knowing which goal it's serving, which is one file read.

**Choice:** 2 — Multiple active goals, interleaved.

---

### Q7: What happens when a step fails?

**Context:** The skill assumes every step succeeds. But tools fail, commands crash, files conflict. The current design has no path for this.

**Options:**
1. Treat failure as a step result — record failure in `completed_steps`, let judge evaluate if it's a pivot or continue. Failure is data, not a separate path.
2. Retry with backoff — retry failed tool call up to N times before recording failure. Handles transient errors without triggering pivot.
3. Immediate halt on failure — any step failure halts the goal. Safest, but overkill.

**Recommendation:** Options 1+2 combined — retry transient failures up to 3 times. If it still fails, record the failure as a step result. The judge decides pivot or continue. The agent doesn't make the halt/pivot decision itself.

**Choice:** Follow recommendation (1+2 combined).

---

### Q8: What structure should `halt_reason` have?

**Context:** When a goal halts, `halt_reason` is a free-form string. But resumption depends on understanding *why* the goal halted. Free-form text is hard to parse programmatically.

**Options:**
1. Free-form string — agent writes whatever explains the halt. Maximum expressiveness, but resumption requires re-interpretation every time.
2. Coded reason + explanation — `halt_code` enum plus `halt_explanation` string. Resumption logic branches on code, explanation gives human context.
3. Two fields replacing one — drop `halt_reason`, add `halt_code` and `halt_explanation`. Cleaner schema, explicit contract.

**Recommendation:** Option 3. The enum makes resumption predictable — "was it achieved?" is one code check, not a text search. Two fields with clear jobs beats one field doing two jobs.

**Choice:** 3 — Two fields replacing one.

---

### Q9: Should there be a max steps limit?

**Context:** The "stuck" detection (3+ steps with no progress → pivot) catches stagnation, but there's no hard ceiling. A runaway loop could burn tokens indefinitely.

**Options:**
1. No hard limit — the judge is the only guardrail. Trusts the judge fully, but a biased judge could loop forever.
2. Configurable max steps — user sets a limit when creating the goal (default: 20). Agent checks `completed_steps.length`. If exceeded, halted with `halt_code: "max_steps_reached"`.
3. Adaptive limit — no hard cap, but at 20 steps agent pauses and asks user: "This goal has taken 20 steps. Continue?"

**Recommendation:** Option 2 — configurable max steps with a sane default. User can override for long goals. Default prevents runaway loops without requiring thought for simple goals.

**Choice:** 2 — Configurable max steps with a default of 20.

---

### Q10: How does the agent pick which goal to work on?

**Context:** When multiple goals are active, the skill says the agent works on "the most recently updated" one. But recency is a weak heuristic.

**Options:**
1. Recency — always continue the most recently updated goal. Simple, predictable. But low-priority goal steals focus from high-priority one.
2. User specifies — agent asks "which goal?" at start of each invocation. Maximum clarity, but breaks autonomous promise.
3. Priority field — user assigns priority (`high`, `medium`, `low`). Agent picks highest-priority active goal. Predictable without user intervention, but adds a field.
4. Default to recency, allow override — by default, continue most recently updated. User can name a different goal to switch to. No extra fields, preserves autonomy.

**Recommendation:** Option 4. Keeps autonomous loop smooth, user can interrupt and redirect. No new schema fields, no mandatory decisions at goal creation.

**Choice:** 4 — Default to recency, allow override.

---

### Q11: What does the judge receive?

**Context:** We decided the judge is a separate model call, but what actually goes into that call?

**Options:**
1. Goal state JSON only — judge evaluates purely from structured data. Maximum consistency, but might miss nuance not in schema fields.
2. Goal state + last step's raw output — judge sees actual evidence (file diff, command output). More accurate, but raw output could be large.
3. Goal state + agent's step summary — judge gets JSON plus brief narrative summary. More context, but filtered through agent's interpretation.

**Recommendation:** Option 2 — judge should see raw evidence, not agent's interpretation. Makes judge independently verifiable.

**Choice:** 3 — Goal state + agent summary. (User preferred the agent's interpretation over raw output.)

---

### Q12: What shape does the judge return?

**Context:** The judge returns a verdict, but the skill doesn't define the shape. Free text is fragile; structured data is deterministic.

**Options:**
1. Structured JSON — judge returns JSON with `verdict`, `reasoning`, `evidence`, `next_step_suggestion`. Predictable, but agent must handle malformed JSON.
2. Fixed-format text — specific text format. More resilient to model quirks, but brittle if model deviates.
3. JSON with fallback — judge prompted to return JSON. If parsing fails, agent treats as `pivot` and logs unparseable output. Handles model failures gracefully.

**Recommendation:** Option 3 — JSON with fallback. Most models will comply, and when they don't, safe default is `pivot` — agent asks user instead of guessing.

**Choice:** 3 — JSON with fallback.

---

### Q13: Where does the judge prompt live?

**Context:** The judge is a separate model call, but what actually goes into that call? The prompt template is the most important piece of the system.

**Options:**
1. Inline in SKILL.md — agent constructs prompt from template in skill. Everything in one place, but adds length to SKILL.md.
2. Separate prompt file — `goal/JUDGE-PROMPT.md` that agent reads and sends as system prompt. Progressive disclosure, SKILL.md stays tight.
3. Hardcoded in agent's logic — prompt constructed from goal state fields. No template file, but inconsistent across runs.

**Recommendation:** Option 2 — separate prompt file. Judge prompt is a reference document, not a step. It belongs behind a pointer. Keeps skill tight and prompt maintainable.

**Choice:** 2 — Separate prompt file.

---

### Q14: How does a user revise a goal mid-loop?

**Context:** We decided goals are refinable with history, but never says *how* a revision happens.

**Options:**
1. Explicit command — user types `/goal revise <id> <new goal>`. Clean, intentional, but user must remember syntax.
2. Natural language detection — user says "actually, the goal should be X" and agent recognizes it. No special syntax, but might misclassify.
3. Agent-initiated — during pivot, agent can propose goal revision. User confirms. Agent can say "your goal is wrong, here's a better one."

**Recommendation:** Option 2. Agent detects revision intent naturally, confirms before applying.

**Choice:** 2 and 3 — Natural language detection AND agent-initiated revisions. Both require user confirmation.

---

### Q15: Should steps have an explicit number?

**Context:** The `completed_steps` array grows over time. When a goal is resumed, the agent needs to know what step it's on.

**Options:**
1. Array index only — step 1 is `completed_steps[0]`. No extra field. Simple, but if steps are ever reordered, numbering shifts.
2. Explicit `step_number` field — each step entry includes `"step_number": 5`. Survives reordering, but redundant field agent must keep in sync.
3. No step numbers anywhere — agent refers to steps by content. More human-readable, harder to reference precisely.

**Recommendation:** Option 1 — array index only. Steps are append-only in practice. Array length IS the step count. Adding explicit number is maintenance with no real benefit.

**Choice:** 2 — Explicit step_number field.

---

### Q16: What does the status line look like?

**Context:** After each step, agent prints a brief status so user can see progress and intervene. Format matters — too much is noise, too little is opaque.

**Options:**
1. One-liner — `Step 3 done: wrote auth handler. Judge: continue.` Maximum brevity, can't see evidence or what's next.
2. Structured block — three lines: step result, verdict, next step. More context, 60 lines in a 20-step loop.
3. Compact with detail on demand — one line by default, expands on request. User can skim, detail available.

**Recommendation:** Option 2. Three lines per step is small price for visibility. In 20-step loop, user scrolls past 60 lines — fine, they can skim. When judge says `pivot`, three-line format makes it immediately clear why.

**Choice:** 2 — Structured block.

---

### Q17: Who maps the judge's `halt` verdict to a `halt_code`?

**Context:** The `halt_code` has `goal_achieved` and `unachievable` as separate values. Both produce `halt` from the judge, but the agent must decide which code to write.

**Options:**
1. The judge — add `halt_code` to judge's return schema. Judge returns `{ "verdict": "halt", "halt_code": "goal_achieved", ... }`. Direct, no interpretation layer.
2. The agent — judge returns `{ "verdict": "halt", "reasoning": "..." }` and agent reads reasoning to decide code. More flexible, but interpretation step could go wrong.
3. The judge with a lookup — judge prompt includes halt_code enum and says "return the matching code." Structured, no interpretation.

**Recommendation:** Option 3. Judge prompt already has decision questions — adding halt_code enum lets judge return code directly. Agent just passes it through.

**Choice:** 1 — Judge returns it directly. (Essentially the same as option 3 — judge includes halt_code in its return.)

---

### Q18: Who creates the first step?

**Context:** When a goal is created, step 4 says "set `current_step` to the first concrete action." But who decides what that first step is?

**Options:**
1. The agent decides — user states goal, agent breaks it down and proposes first step. Fast, autonomous, but might misunderstand.
2. The user provides — agent asks "what's the first step?" and user answers. Accurate, but user does the planning work.
3. Collaborative — agent proposes first step and rough plan, user confirms or corrects. Agent does thinking, user does vetting.

**Recommendation:** Option 3 — collaborative. Agent proposes, user approves before execution. Catches misunderstandings, keeps user in control of direction.

**Choice:** 3 — Collaborative.

---

### Q19: Should this skill be model-invoked or user-invoked?

**Context:** The original prompt described two invocation types: model-invoked (agent can fire autonomously) vs user-invoked (only fires when user types name).

**Options:**
1. Model-invoked — agent triggers goal loop autonomously when it detects multi-step work. Description stays in system prompt (context load), but agent can reach skill without user typing `/goal`.
2. User-invoked — only fires when user explicitly types `/goal`. Zero context load, but user must remember skill exists.
3. Hybrid — model-invoked with minimal description. Agent recognizes `/goal` and "work toward" but doesn't load heavy description every turn.

**Recommendation:** Option 1 — model-invoked. Skill is most useful when agent can recognize "this is a multi-step task" and start loop on its own. User shouldn't have to type `/goal` for every multi-step task.

**Choice:** 2 — User-invoked. (User wanted explicit trigger only.)

---

### Q20: How does the user delete a goal?

**Context:** The skill says "Never delete active goals" — halting is the safe alternative. But halted and completed goals pile up forever.

**Options:**
1. Manual file deletion — user deletes `goals/<id>.json` themselves. Skill never deletes files. Simple, but requires user to know file path.
2. Agent deletes on request — user says "delete goal fix-auth-timeout" and agent removes file. Convenient, but accidental deletion irreversible.
3. Soft delete — agent moves file to `goals/.archived/<id>.json`. Hidden from listings but recoverable. Permanent deletion is separate act.

**Recommendation:** Option 3 — soft delete. Safest option. Goals disappear from listings but aren't gone. User can recover if they change their mind.

**Choice:** 3 — Soft delete.

---

### Q21: What happens when the agent disagrees with the judge?

**Context:** The judge might return `continue` when the agent believes something went wrong. Agent has local knowledge the judge doesn't.

**Options:**
1. Judge always wins — agent obeys verdict regardless. Simple, predictable, but bad judge decision sends loop wrong path.
2. Agent can object — agent says "I disagree because X" and proposes different verdict. User decides. More accurate, adds negotiation step.
3. Agent can flag but not override — agent appends `disagreement` note to goal state, but still obeys verdict. Flag is informational, shows up in evaluation history.

**Recommendation:** Option 3. Judge is authority for loop control, but agent can flag concerns. Keeps loop moving while creating paper trail.

**Choice:** 3 — Agent can flag but not override.

---

### Q22: What happens on user interruption?

**Context:** Loop runs autonomously, user can interrupt at any time. What happens to the in-progress step?

**Options:**
1. Step is lost — agent stops immediately, step abandoned. Partial file changes remain. Resume picks up from `current_step` again.
2. Step completes, then stops — agent finishes current step, saves result, then stops. Cleaner state, but user might not want step to finish.
3. Agent checkpoints — agent writes partial result to goal state before stopping. On resume, agent can pick up where it left off.

**Recommendation:** Option 1 — step is lost. Agent stops as fast as possible. Partial step with partial changes is worse than clean restart.

**Choice:** 3 — Agent checkpoints. (User wanted partial work preserved for resumption.)

---

### Q23: What happens when a goal file is unreadable?

**Context:** System depends entirely on JSON files in `goals/`. If file is corrupted, missing, or manually edited badly, whole loop breaks.

**Options:**
1. Fail loudly, don't guess — agent reports error, refuses to proceed. Safe, but user has to fix file manually.
2. Attempt recovery — agent tries to reconstruct goal from filename and partial data. Risky — might reconstruct incorrectly.
3. Fail loudly with guidance — agent reports error and tells user how to fix it. Safe, and user knows exactly what to do.

**Recommendation:** Option 3. Don't guess at recovery — corrupted state is worse than no state. But don't just say "error" either. Show user what's wrong and what valid file looks like.

**Choice:** 3 — Fail loudly with guidance.

---

### Q24: What does the agent display on resume?

**Context:** When user resumes a goal, they need to know where things stand. What does agent show before starting loop again?

**Options:**
1. Just the next step — "Resuming goal fix-auth-timeout. Next step: write unit tests." Minimal, user might not remember before.
2. Full summary — goal, current step, last 3 completed steps with results, last verdict. Complete context, lot of text.
3. Structured resume block — compact display: goal, step count vs max, status, last step result, next step. Three to four lines.

**Recommendation:** Option 3 — structured resume block. Enough context to orient, not so much that it's noise.

**Choice:** 3 — Structured resume block.

---

### Q25: Which file is the source of truth for the judge?

**Context:** Two files define the judge's behavior: `EVALUATION.md` and `JUDGE-PROMPT.md`. They overlap but aren't identical. Duplication risk — one will drift.

**Options:**
1. JUDGE-PROMPT.md — this is what the judge actually receives. It's the contract. EVALUATION.md becomes human-readable summary.
2. EVALUATION.md — this is the spec. JUDGE-PROMPT.md is generated from it. EVALUATION.md defines rules; JUDGE-PROMPT.md delivers them.
3. Merge them — one file does both jobs. Agent reads EVALUATION.md and sends as judge prompt. No duplication, but serves two audiences.

**Recommendation:** Option 1. JUDGE-PROMPT.md is the contract — it's what the judge sees, so it must be right. EVALUATION.md becomes documentation. If they diverge, prompt wins.

**Choice:** 1 — JUDGE-PROMPT.md.
