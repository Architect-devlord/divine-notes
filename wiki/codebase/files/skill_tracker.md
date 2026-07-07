---
type: file
status: ingested
---
# skill_tracker.py

💡 **Role**: the smallest of these four — scores each real attempt at a focused task against what deliberation imagined would be best, purely by comparing the agent to its own recent past. [[reward-and-learning-stack]] already covers `record_attempt()`'s core comparison and `get_weakest_task()`'s role in focus-task resolution; this page adds the three read-only methods the topic page doesn't mention and one removed-dead-code note.

## Imports

`logging`, `torch`, `torch.nn`, `typing` — all external. No internal project imports; `continual_learner` arrives as a constructor argument and is stored but never called into from this file.

## Classes

### `SkillTracker`

**Methods:**

- `__init__(continual_learner)` — stores a reference (currently unused elsewhere in the class — see below) and an empty per-task stats dict.
- `record_attempt(task_label, obs_vector, action_taken, imagined_target)` — MSE between the action actually taken and deliberation's imagined target, appended to a rolling 50-entry window per task. `obs_vector` is accepted and converted to a tensor but never actually used in the loss calculation — kept as a parameter for API symmetry and so a real embedding-based score could be added later without changing the call signature. Returns `improving` (`True` if this loss beat the previous one — a two-sample comparison, not a trend over the window) and `score` (`1 − avg_loss`, clamped to `[0, 1]`).
- `get_all_scores()` — `1 − avg_loss` for every task with at least one recorded attempt.
- `get_best_task()` / `get_weakest_task()` — argmax/argmin over `get_all_scores()`.
- `summary()` — all tasks sorted by attempt count (most-practiced first), each with score, attempt count, and a two-sample trend arrow.

No async methods.

**Removed-dead-code note, from the file's own comment**: an earlier version computed an `_embedding` from the continual-learning policy net's first layer on every single attempt and never used it for anything — a pure wasted forward pass. Removed; `self.cl` (the stored `continual_learner` reference) is consequently unused anywhere in this file's current logic.

## Problems (faced by traditional AI systems / LLMs)

Skill-progress tracking usually needs either an external grader or a fixed benchmark to compare against, neither of which exists for an open-ended agent practicing a self-chosen task with no ground truth for "correct." A related, narrower problem: code that computes an intermediate value (an embedding, a feature) "just in case it's useful later" tends to survive long after the thing it was for is gone, quietly costing compute on every call for no behavioral benefit.

## Solutions

Comparison against the agent's own imagined target (from [[imagination-and-planning]]'s deliberation) sidesteps the external-grader problem entirely — there is no teacher anywhere in this file, only "did this attempt get closer to what I myself predicted would work." The dead-embedding problem gets the direct fix: remove the unused computation rather than leave it as latent technical debt, keeping the parameter it would have used for API-shape stability alone.

## Files Required

None from within this codebase — `continual_learner` is passed in and stored but never called into.

## Files Used In

- `agent.py` — constructs `SkillTracker` from `self.continual_learner` (see [[agent-runtime]] and [[agent]]).
- `cognitive_loop.py` — `_record_skill_attempt()` is the actual call site for `record_attempt()`; `_resolve_active_focus_task()`'s Path 1 fallback calls `get_weakest_task()` (see [[cognitive_loop]]).