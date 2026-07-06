---
type: file
status: ingested
---

# planner.py

💡 **Role**: multi-step sequence planning via world-model imagination. [[imagination-and-planning]] already covers this file's design philosophy, `ImagineResult`'s four properties, the full `scoring_fn` menu, and `imagine_death()` in narrative depth — this page is the mechanical reference (every class/function, one line each) plus the implementation details that page doesn't get into.

## Imports

`random`, `numpy`, `torch`, `typing` — external. One module-level internal import: `from ai_core.brain_core import BrainCore` (the type `CognitivePlanner.__init__` expects). Two more, lazy, inside `_run_imagination()` only: `from ai_core.world_model import _build_observation_from_context` and `from ai_core.brain_core import _action_to_vector` — see [[brain-core]] and [[world-model]].

## Module-level constants

- `DEFAULT_TEMPLATES` — the 10 seed action dicts. See [[imagination-and-planning]] for the full list and its role as a bootstrap seed rather than a ceiling.
- `ACTION_TYPE_INDEX` — the shared one-hot encoding table, 11 entries (10 template types plus `sleep`, which has no matching `DEFAULT_TEMPLATES` entry).

## Classes

### `ImagineResult`
Holds one imagined sequence's full, unfiltered output. See [[imagination-and-planning]] for what each property means and why nothing is clipped before the agent sees it.

**Methods:**
- `total_reward` (property) — discounted cumulative reward, γ=0.95.
- `peak_death_probability` (property) — worst single-step death probability.
- `expected_survival` (property) — probability of surviving every step.
- `dies_in_imagination` (property) — `True` if any step exceeds 50% death probability.
- `summary()` — the above four plus raw per-step rewards/death-probs and the sequence's action types, as a plain dict (JSON-ready for sending over a WebSocket or logging).

### `CognitivePlanner`
The planner itself — see [[imagination-and-planning]] for `scoring_fn`'s design and the full menu of example risk personalities.

**Methods:**
- `generate_plan(obs, memory, horizon, n_trials, context)` — the public entry point; routes to one of the two private planners below depending on whether a world model and context are both available.
- `imagine_sequence(sequence, context)` — imagine one specific, caller-provided sequence rather than a random one. Returns `None` if no world model is attached.
- `imagine_death(context, n_samples)` — see [[imagination-and-planning]].
- `imagine_many(n, horizon, context)` — like `imagine_death()` but returns all `n` random rollouts unsorted, for surveying the outcome space rather than specifically hunting worst-case ones.
- `set_scoring_fn(fn)` — swaps the risk-personality function at runtime.
- `add_template(action)` — appends a new template if not already present; the hook [[emergent_templates]]'s graduated skills and `god_controls.py`'s god abilities both use.
- `get_best_single_action(context)` — the single highest-scoring template right now, no multi-step rollout involved.
- `_plan_with_world_model(obs, memory, horizon, n_trials, context)` — the real planner: narrows to the top 5 templates by single-step value, then samples `n_trials` random sequences from just those 5, scoring each with a full `_run_imagination()` rollout when it succeeds, or a summed single-step fallback when it doesn't. Adds a novelty bonus and a repetition penalty (see `_novelty_bonus()` below) before picking the best.
- `_run_imagination(sequence, context)` — builds the initial observation, encodes every step of the sequence via `_action_to_vector()` (so a discovered skill's real vector is used instead of forcing a one-hot guess), and calls `wm.imagine()` under `torch.no_grad()`. Wrapped in a broad `try/except` — any failure (a malformed context, a world-model shape mismatch) is logged as a warning and returns `None` rather than raising, which is exactly the signal `_plan_with_world_model()` treats as "fall back to the table method for this trial only," not a hard failure of planning as a whole.
- `_plan_with_table(obs, memory, horizon, n_trials, context)` — the no-world-model fallback: identical sampling/novelty/repetition logic to `_plan_with_world_model()`, but every candidate sequence is scored by summing `brain.predict_value_of_action()` per step instead of running an actual imagined rollout.
- `_score_templates(context)` — single-step value for every current template, as `(score, template)` pairs.
- `_novelty_bonus(seq, memory)` — for each action in the sequence, counts how many times that exact action *type* appears in `memory.events` and scores `1/(1+count)` (so a never-tried type scores 1.0, a heavily-repeated one approaches 0), averaged across the sequence. Silently treats a missing or malformed `memory.events` as zero prior occurrences rather than raising.

No async methods or functions anywhere in this file — every rollout, including the world-model call, runs synchronously.

## Problems (faced by traditional AI systems / LLMs)

Classical planning (search over a fixed action set, scored by a fixed objective) can't represent an agent that might reasonably want different things at different points in its life — risk tolerance is usually baked into the reward function or the search heuristic itself, not exposed as something that can change. Separately: most RL agents that use imagined rollouts at all quietly discard or clip catastrophic predicted outcomes before scoring, on the assumption that death/failure trajectories are noise to be filtered rather than signal to be learned from.

## Solutions

`scoring_fn` externalizes "what counts as good" into a swappable function instead of a fixed reward shape — the same imagined rollout can be evaluated as reckless, cautious, or curious about danger depending only on which function is currently attached, with no change to how the rollout itself is generated. `ImagineResult` never filters: every step's predicted reward and death probability reaches the scoring function and, from there, whatever's inspecting it — the "clip catastrophic outcomes" step other systems take by default simply doesn't exist here.

## Files Required

- `brain_core.py` — `BrainCore` (constructor type) and `_action_to_vector()` (used inside `_run_imagination()`). Not yet given its own file-level page — see [[brain-core]] for the narrative treatment in the meantime.
- `world_model.py` — `_build_observation_from_context()`, and `wm.imagine()` on whatever `WorldModel`/`EnsembleWorldModel` instance is attached. Not yet given its own file-level page — see [[world-model]].
- [[emergent_templates]] — indirectly: `add_template()` is the hook graduated skills arrive through, though this file has no direct import of it.

## Files Used In

- [[cognitive_loop]] — calls `generate_plan()` to drive multi-step plan execution.
- `god_controls.py` — calls `add_template()` to register god-specific abilities as candidate plans. Not yet given its own file-level page.
