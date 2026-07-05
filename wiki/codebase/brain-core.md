---
type: system
status: ingested
---

# Brain Core

💡 **What this is**: `ai_core/brain_core.py`'s `BrainCore` — the piece [[agent-runtime]] and [[cognitive-loop]] both lean on constantly without it having had its own page until now. Sits between raw perception and action; owns the fast/slow evaluation split that makes deliberation affordable at all (see [[hardware-requirements]] for what deliberation costs).

## Design rules, straight from the file's own header

- `evaluate_event()` is always fast. Never blocks on torch.
- `deliberate()` is always explicit — the cognitive loop calls it only when it's decided the situation is worth deep thought.
- There is no `GodBrainExtension`. **Gods use the same brain as everyone else** — personality weights and reward-system configuration are what make a god different, not a different code path. See [[god-abilities]] for what *does* differ (the action space and ability set).
- `BrainCore` owns its world-model connection via `set_world_model()` — nothing patches `evaluate_event` from outside.

## PatternRecognizer — the novelty engine

A lightweight online tracker across five pattern types (`behavior`, `visual`, `audio`, `language`, `state`). `observe_pattern()` hashes the observation (MD5 over sorted JSON — a fixed bug: the previous hash scheme joined rounded-float strings, and distinct observations with similar shapes routinely collided, corrupting the novelty signal that gates *all* deliberation), and returns novelty as `1 / (1 + times_seen_before)` — genuinely new things score near 1.0, familiar things decay toward 0. Also tracks pattern-to-pattern transitions, so `predict_next_pattern()` can answer "what usually follows this."

## The fast path — `evaluate_event()`

Runs every cognitive cycle, no world model involved. Two routes:

- **RewardSystem attached** (the normal case): builds obs/action tensors from the agent's last cached observation and routes through `reward_system.compute_reward()` + `apply_signal()` — full personality-weighted scoring. A real fixed bug lived here: obs/action tensors weren't being passed at all, so RND/ICM's intrinsic curiosity was silently always 0.0 for anything that wasn't a full RL step — brain-level events like chat or a sound had no intrinsic motivation attached to them at all until this was fixed.
- **No RewardSystem** (fallback): a hand-rolled scorer — drive reward (health/hunger deltas, danger, success) + curiosity-weighted novelty + a learned value-table lookup − a prediction-error penalty.

## The gate — `should_deliberate()`

Returns `False` fast when there's no world model, when on cooldown (default 2.0s), or when the situation is routine. Fires `True` when novelty ≥ 0.55 **or** urgency ≥ 0.60. This is the entire reason deliberation is affordable — it doesn't run every cycle, only when something's actually interesting.

## `should_browse()` — a second, parallel gate

Not previously documented: the same brain also gates autonomous web browsing, independently of deliberation. `browse_desire = curiosity×0.45 + openness×0.25 + novelty×0.20 + (1−conscientiousness)×0.10`, boosted +0.15 if recent memory mentions a URL or an unanswered question, suppressed entirely above urgency 0.70, rate-limited to once per 60s, and gated stochastically (`random() < browse_desire`, not a hard threshold) specifically so browsing doesn't fall into a mechanical period. See [[perception-and-actuation]] for the browser tool itself.

## The slow path — `deliberate()`

Called only when `should_deliberate()` said yes. `horizon=12, n_trials=40` (fixed up from `horizon=3, n_trials=15` — the original was "too myopic" per the fix comment) — confirms the 480-forward-pass cost cited in [[hardware-requirements]]. Builds a context from perception, resolves candidate actions (from [[imagination-and-planning]]'s `DEFAULT_TEMPLATES` if none given), and scores each candidate by imagining short rollouts:

- If a world model is attached, `_deliberate_with_world_model()` runs the actual imagination (see [[world-model]]).
- **Uncertainty penalty**: if an ensemble world model is attached, each candidate's score is reduced by `uncertainty_lambda × ensemble_std × (1 + candidate_novelty)`. `uncertainty_lambda = max(0.05, 0.25 − 0.2×boldness + 0.2×neuroticism)` — bold agents barely penalize uncertain bets, neurotic ones penalize them heavily. This is the concrete mechanism behind the "prevents the policy from exploiting confident-but-wrong predictions" claim in [[reward-and-learning-stack]] — previously described there only at a high level; this is the actual formula.
- Falls back to `_deliberate_with_table()` (the learned value table) if no world model, or if world-model deliberation throws.
- Returns a `DeliberationResult`.

## DeliberationResult

- `best_action` / `best_action_vector` / `best_score` — the top-ranked candidate, as a dict, as a raw one-hot-style numpy vector (for [[reward-and-learning-stack]]'s `SkillTracker`, fully self-referential — no external teacher), and as a score.
- `all_scored_actions` — every candidate's `(score, action_vector)` pair, kept in the same order as `ranked_actions` — this is what GRPO actually trains on.
- `should_abort_current_plan(running_plan_score)` — `True` if urgency ≥ 0.75, or if the best fresh option beats the running plan by more than 20%. This is the exact mechanism [[cognitive-loop]]'s mid-plan interruption logic calls.

## A quietly important fixed bug

`BrainCore` used to *also* eagerly construct a `LanguageIntelligence` instance in `__init__`, directly contradicting its own docstring, which said language should be attached later via `add_language_to_brain()`. Since that function unconditionally overwrites `self.language` moments later in `NPCAgent.__init__` anyway, the first instance was simply discarded — every single agent spawn was building and immediately throwing away a full duplicate transformer, tokenizer, and AdamW optimizer. Fixed by leaving `self.language = None` here; `add_language_to_brain()` (see [[language-system]]) is now the sole construction path.
