---
type: file
status: ingested
---

# brain_core.py

💡 **Role**: sits between raw perception and action — owns the fast/slow evaluation split, deliberation, and (new in commit `8839776`) the shared action-vector encoding every candidate-scoring site now uses. [[brain-core]] already covers `PatternRecognizer`, `evaluate_event()`, `should_deliberate()`, `should_browse()`, `deliberate()`, and `DeliberationResult` in narrative depth, including three fixed-bug stories — not repeated here beyond a one-line pointer per method. This page's real additions: the module-level `_action_to_vector()` helper (added this commit, not yet on the topic page) and the ~15 methods the topic page doesn't mention at all.

## Imports

`__future__.annotations`, `logging`, `threading`, `collections` (`defaultdict`, `deque`), `time.time`, `typing`, `numpy` — all external/stdlib. No internal project imports at module level; two lazy internal imports inside specific methods: `from ai_core.planner import ACTION_TYPE_INDEX` (inside `_action_to_vector()`) and `from ai_core.world_model import _build_observation_from_context` (inside `_world_model_action_value()`) — see [[imagination-and-planning]] and [[world-model]]. The lazy import of `ACTION_TYPE_INDEX` specifically avoids a circular import: `planner.py` imports `BrainCore` from this file at its own module level, so this file can't import `planner.py` back at module level too.

## Module-level function

- `_action_to_vector(action, action_dim)` — the shared helper `demo-features-explaination.md` centers on. Uses a candidate's own `action_vector` field when present (a discovered skill's real centroid from [[emergent_templates]]), padding or truncating to `action_dim`; otherwise falls back to the original `ACTION_TYPE_INDEX` one-hot lookup unchanged. Every one-hot construction site in this file and in [[imagination-and-planning|planner.py]] now calls this instead of building the vector inline.

## Classes

### `PatternRecognizer`
See [[brain-core]] for the novelty formula and the MD5 hashing bug fix.

**Methods:**
- `observe_pattern(pattern_type, pattern_data)` — records an observation and its transition from the previous one.
- `get_novelty(pattern_type, pattern_data)` — `1/(1+times_seen)`.
- `predict_next_pattern(pattern_type)` — most common transition seen so far from the current pattern.
- `get_pattern_stats(pattern_type)` — counts and top transitions for one pattern type.
- `_hash_pattern(data)` — MD5 over sorted JSON.
- `_related(pt, h, limit)` — nearby pattern hashes by transition frequency.

### `DeliberationResult`
See [[brain-core]] for `all_scored_actions`' role in GRPO training and `should_abort_current_plan()`'s exact thresholds.

**Methods:**
- `best_action` / `best_action_vector` / `best_score` (properties) — top-ranked candidate as dict, vector, and score.
- `should_abort_current_plan(running_plan_score)` — interruption check [[cognitive_loop]] calls.
- `summary()` — JSON-ready dict of the above.

### `BrainCore`
**Methods not covered on [[brain-core]]:**
- `__init__(agent_ref)` — sets up the value table, forward model, `PatternRecognizer`, a 10,000-entry continual-experience deque, and deliberation/browsing thresholds as plain instance attributes. `self.language` is deliberately left `None` here (see [[brain-core]]'s fixed-bug note) rather than constructed.
- `world_model` (property + setter) — thread-safe swap via `self._wm_lock`, so a world model can be attached or replaced from a different thread than whatever's currently reading it mid-deliberation.
- `set_world_model(wm)` — the one sanctioned way to attach a world model; logs whether it has a VAE.
- `set_reward_system(rs)` — attaches a `RewardSystem`.
- `predict_value_of_action(action, context)` — the single-step counterpart to `deliberate()`'s multi-step scoring: tries the world model first if both it and `context` are available, falls back to the value table on any exception.
- `_table_action_value(action)` — sums `forward_model` transition probabilities weighted by each outcome's learned average value.
- `_world_model_action_value(action, context)` — builds an observation, encodes the action via `_action_to_vector()`, and reads the predicted reward straight from the world model's output.
- `process_language_input(text, context, speaker_id)` — thin wrapper around `LanguageIntelligence.process_input()`; a recently-fixed call site (`speaker_id` was missing entirely, which would have raised once the underlying method started requiring it for familiarity tracking).
- `get_language_progress()` — current language stage and vocab size, for status reporting.
- `get_continual_buffer(task, limit)` — filtered/truncated read of `continual_buffer`.
- `switch_task(new_task)` — updates `current_task` and timestamps it in `task_labels`.
- `get_pattern_summary()` — `get_pattern_stats()` for every pattern type at once.
- `_classify_event_pattern_type(event)` — buckets an event into one of `PatternRecognizer`'s five types by tag/type-string matching.
- `_drive_reward(payload)` — health/hunger deltas plus danger/success flags, summed into a scalar — the drive-reward term `evaluate_event()`'s no-RewardSystem fallback uses.
- `_lookup_learned_value(event)` — average historical reward for this `(type, first tag)` key.
- `_prediction_error(event)` — `1 − forward_model`'s predicted probability of the tag that actually occurred.
- `_update_learning(event, payload, reward)` — updates both the value table and the forward model's transition-frequency counts (renormalized to probabilities) after every evaluated event.
- `_store_continual_experience(event, reward, context)` — appends one entry to `continual_buffer`, tagged with the current task and a timestamp.
- `_perception_to_context(perception)` — reshapes a cognitive-loop perception dict into the flatter `{health, hunger, emotions, novelty, urgency}` shape `_build_observation_from_context()` expects.
- `_action_novelty_bonus(action)` — `1/(1+total_uses)` against the forward model's use count, or `1.0` if never attempted — the same shape as `planner.py`'s `_novelty_bonus()` but keyed off the forward model instead of raw memory events.
- `_reward_to_emotion_delta(reward, event)` — converts a scalar reward into `{joy, fear, surprise}` deltas; used as a fallback path when no `RewardSystem` is attached.

No async methods anywhere in this file.

## Problems (faced by traditional AI systems / LLMs)

Any agent architecture that deliberates via world-model rollouts faces a basic affordability problem: full imagination is expensive (see [[hardware-requirements]] for the actual forward-pass cost here), so running it on every tick either caps how fast the agent can act or wastes most of that compute on routine, uninteresting moments. Separately, systems with more than one place that turns an action into a trainable vector tend to accumulate silently-inconsistent encodings over time — exactly the one-hot-collapse bug the demo file traces across three separate call sites before this commit's fix.

## Solutions

The fast/slow split ([[brain-core]]'s `evaluate_event()`/`should_deliberate()`) is the affordability answer — cheap scoring runs constantly, expensive imagination runs only when novelty or urgency crosses a threshold. `_action_to_vector()` is the consistency answer for the second problem: one function, called from every site in this file and in `planner.py` that used to build a one-hot vector inline, so a discovered skill's real vector is honored everywhere at once rather than needing three separate fixes kept in sync by hand.

## Files Required

- `planner.py` — `ACTION_TYPE_INDEX` (lazy import, inside `_action_to_vector()`). Not yet given its own file-level page — see [[imagination-and-planning]] for the narrative treatment in the meantime.
- `world_model.py` — `_build_observation_from_context()` (lazy import, inside `_world_model_action_value()`), and whatever `WorldModel`/`EnsembleWorldModel` instance is attached via `set_world_model()`. Not yet given its own file-level page — see [[world-model]].
- [[emergent_templates]] — indirectly: `_action_to_vector()`'s `action_vector` branch exists specifically to honor this module's output, though there's no direct import of it here.
- `brain_language.py` — `LanguageIntelligence`, attached externally via `add_language_to_brain()`, not constructed in this file. Not yet given its own file-level page — see [[language-system]].

## Files Used In

- `agent.py` — constructs `BrainCore` and wires up its sub-systems. Not yet given its own file-level page — see [[agent-runtime]] for the narrative treatment in the meantime.
- [[cognitive_loop]] — calls `evaluate_event()`, `should_deliberate()`, `deliberate()`, and reads `DeliberationResult` every cycle.
- `planner.py` — imports `BrainCore` (as a type) and `_action_to_vector()`. See [[imagination-and-planning]] for the narrative treatment.
- `skill_tracker.py` — `DeliberationResult.best_action_vector` feeds `SkillTracker`. Not yet given its own file-level page.
- `self_supervised_trainer.py` — `all_scored_actions` is what `grpo_update()` (defined here, on the `TransformerPolicy`) trains on. Not yet given its own file-level page — see [[reward-and-learning-stack]] for the narrative treatment.
