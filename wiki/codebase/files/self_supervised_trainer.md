---
type: file
status: ingested
---
# self_supervised_trainer.py

💡 **Role**: two mostly-unrelated things that happen to share a file — `SelfSupervisedTrainer` (trains the world model from the agent's own experience, prediction error as the signal) and `grpo_update()` (the actual policy-gradient step deliberation's scored actions feed into). [[reward-and-learning-stack]] already covers both at the concept level, including this file's own "GRPO" section; this page is the mechanical detail and two fixed bugs neither summary spells out.

## Imports

`logging`, `random`, `torch`, `torch.nn`, `torch.nn.functional`, `typing`, `numpy` — all external. No internal project imports; `world_model`/`brain`/`emotion_system` and `model` (for `grpo_update`) all arrive as arguments, never imported directly.

## Classes

### `SelfSupervisedTrainer`

**Methods:**

- `__init__(world_model, brain, emotion_system, device)` — deliberately does _not_ create its own optimizer. An earlier version did (`AdamW` over the world model's parameters), which — since `WorldModel` already owns and calls its own internal optimizer inside `train_step()` — would have applied two independent, conflicting gradient updates to the same parameters every step, corrupting convergence. Training now goes entirely through the world model's own `train_step()`.
- `train_step(experience_buffer)` — samples up to 32 transitions (needs at least 16 in the buffer, and at least 8 with all three of `obs_vector`/`action_vector`/`next_obs_vector` present, or returns `None`), and trains the world model to predict `next_obs` from `(obs, action)`. **A real fixed bug, not on the topic page**: the original call site was `world_model.predict()`, a method that doesn't exist on `WorldModel` — this had been silently failing on _every single call_, caught by a broad `except`, meaning the world model had never actually been trained by this path in any deployed agent. Fixed to call `WorldModel.train_step()` with the batch dict shape it actually expects (`proprio`/`action`/`reward`/`termination`/`next_state`, each unsqueezed to a length-1 sequence). Also resolves `self.wm` lazily from `brain.world_model` if it was `None` at construction time, specifically because `SelfSupervisedTrainer` gets built in `NPCAgent.__init__` slightly before the world model does — catching that ordering gap without needing to reorder the constructor itself. Feeds the resulting scalar loss to the emotion system as `mean_surprise`: curiosity+joy if healthy, fear (and curiosity suppression) if health is below 8 — approximated from the aggregate batch loss rather than a true per-transition value, noted in the code as good enough for now.
- `get_stats()` — step count and 100-step rolling average loss.

No async methods — this is the same pattern as the other three files in this batch: everything here is a synchronous, blocking torch call.

## Module-level function

- `grpo_update(model, scored_actions, obs, lr)` — **the actual gradient step**, not attached to any class (monkeypatched onto a policy instance elsewhere, per the file's own module docstring, so it can be called as if it were a method without editing `rl/policy.py` directly). Needs at least 4 scored actions (for meaningful relative comparison); computes advantages as `(score − mean) / std`, then derives log-probabilities through each policy's _actual_ forward path rather than SB3's generic one.
    
    **A real fixed regression, not on the topic page**: this function previously called `policy.evaluate_actions()` — an SB3 base-class method neither `TransformerPolicy` nor `GodTransformerPolicy` actually override. Both policies replace `_build_mlp_extractor()` with `self.features_extractor` and bypass `_get_action_dist_from_latent()` entirely in their own custom `forward()`/`_predict()` — so the inherited base method would either crash on a shape mismatch or silently compute log-probs through a completely different computational path than the one that actually produced the scored actions. The fix reconstructs the distribution manually: run `policy.forward()` for the mean, then assemble a matching std either from `policy.log_std` (`TransformerPolicy`) or by concatenating `log_std_base`/`log_std_params` and padding with zeros for the two discrete ability dims (`GodTransformerPolicy`) — then a `Normal` distribution over that mean/std is what actually produces `log_prob()`. **Confirmed, 2026-07-05**: `communication_protocol.py`'s `_grpo_policy_update()` is not a second implementation — it's a thin, guarded call into `agent.policy.grpo_update()`, itself this exact function monkeypatched onto the live policy instance. One verified implementation, called from two trigger sites, not two implementations. See [[communication_protocol]] for the full account, including why those two trigger sites needed a guard against firing on the same signal twice.
    

## Problems (faced by traditional AI systems / LLMs)

Two distinct, well-known failure modes. First: any training loop with more than one thing capable of stepping the same parameters (here: a wrapper's own optimizer alongside a model's already-internal one) risks the two updates fighting each other silently — nothing crashes, convergence just quietly gets worse. Second: any RL codebase with two independently-evolving custom policy classes will eventually accumulate methods that call a slightly-wrong-for-this-architecture parent-class implementation instead of the actual overridden path — this is exactly the shape of bug static analysis tools struggle to catch, since the base method exists and is callable, it just isn't the code that actually runs during real training.

## Solutions

The optimizer conflict is solved by ownership discipline: exactly one thing (`WorldModel` itself) owns the optimizer for its own parameters, full stop. The evaluate_actions() problem is solved by tracing the real forward path for each specific policy class rather than trusting a generic base-class method to be doing the right thing just because it has a plausible name and doesn't raise an exception on its own.

## Files Required

None from within this codebase — `world_model`, `brain`, `emotion_system`, and `model` all arrive as constructor/function arguments.

## Files Used In

- `agent.py` — constructs `SelfSupervisedTrainer`; `decide()`'s policy call is what `grpo_update()` trains against indirectly (see [[agent]] and [[agent-runtime]]).
- `cognitive_loop.py` — `_continual_learning_worker()` calls `train_step()` and, separately, `grpo_update()` using the most recent deliberation's `all_scored_actions` (see [[cognitive_loop]]).
- `world_model.py` — `WorldModel.train_step()` is the actual training call `train_step()` here delegates to entirely (not yet given its own file-level page — see [[world-model]]).
- `rl/policy.py` — `grpo_update()` is monkeypatched onto `TransformerPolicy`/`GodTransformerPolicy` instances from here, not defined on the classes themselves.