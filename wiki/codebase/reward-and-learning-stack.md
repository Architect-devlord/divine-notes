---
type: system
status: ingested
---
# Reward & Learning Stack

💡 **What this is**: the cluster of systems that actually make an agent get better over time — imagination (WorldModel), motivation (RewardSystem), and three different flavors of learning stacked on top of each other (policy gradient via GRPO, self-supervised surprise, and Avalanche-based continual learning). Dense, but these six pieces are tightly coupled and mostly don't make sense read in isolation.

## WorldModel — the imagination engine

A 6-layer multimodal Transformer, ~10M parameters (`ai_core/world_model.py` — full architecture now covered in [[world-model]]). Two configurations:

- **`EnsembleWorldModel`** (5 members) — the default (`WorldModelConfig.use_ensemble=True`). Computes both a predicted next-state mean _and_ std across the ensemble, giving deliberation access to epistemic uncertainty.
- **Plain `WorldModel`** — single model, no uncertainty signal, used if ensembling is disabled.

**Why the ensemble matters, concretely**: this was a real, fixed bug. `EnsembleWorldModel` existed in code and was fully implemented, but was never actually instantiated anywhere — every agent got a plain single `WorldModel` regardless of config default. Without uncertainty, GRPO (below) trains entirely against imagined rollout scores, which means the policy can learn to seek out states where the WorldModel makes _confident-sounding but wrong_ predictions — exactly the model-exploitation failure mode that uncertainty-penalized deliberation (PETS/MBPO-style) exists to prevent. Fixed by actually constructing the ensemble and wiring its `.models[0]` up as the reference model everything else expects.

**Deliberation cost**: at `horizon=12`, `n_trials=40`, one deliberation event is **480 WorldModel forward passes**. See [[hardware-requirements]] for what that costs in practice.

`WorldModelReplayBuffer` (capacity 100,000, sequence_length 64) and `WorldModelTrainer` (batch_size 16) handle its own training loop, separate from the RL policy's training. See [[world-model]] for the full architecture (encoders, transformer stack, VAE, the ensemble's uncertainty mechanism, and an unrelated discovery — this same file also hosts the Electron frontend's Mental Matrix backend).

## RewardSystem — where reward actually comes from

Constructed with `obs_dim`, `action_dim`, `personality`, `emotion_system`, `use_rnd=True`, `use_icm=True`. Not a single scalar function — several distinct reward components combine:

- **RND (Random Network Distillation)** and **ICM (Intrinsic Curiosity Module)** — the two intrinsic-curiosity mechanisms, both optional flags but both on by default. Their networks are what get saved/restored in the BrainCapsule (see [[memory-and-braincapsule]]).
- **`familiarity_r`** — a repeat-visitor bonus keyed by `speaker_id`, added specifically so a stable per-installation visitor id (sent from the web/Electron frontend) lets an agent build genuine familiarity with someone it's talked to before, rather than treating every chat as a stranger.
- **`evidence_r`** — an openness-scaled reward for web-checking behavior, computed from `claim_support_delta` (how much a specific page visit changed what the agent can find on the current conversation topic, measured by memory-search hit count before/after). Deliberately symmetric — a page that _contradicts_ what was claimed earns the same credit as one that confirms it, because this rewards the act of checking, not being right.
- **Breeding reward** — see [[breeding-system]]; fires through this same system, hard-capped at 0.15.

Every agent's `initialize_reward_system()` call is eager (see [[agent-runtime]]) specifically so `brain.reward_system` is never `None` — a bare fallback `obs_dim=50` default exists in the signature but is essentially dead code now; the real call site always passes the agent's actual 128/13-or-18 dims explicitly (a fixed bug — the stale 50-dim default used to be what actually got used, which crashed the moment a real 128-dim observation hit RND/ICM's first layer).

## Policy — two sizes, one base

`TransformerPolicy` (`BASE_DIM=13`) for ordinary agents, `GodTransformerPolicy` (`TOTAL_DIM=18`) for god agents — see [[agent-runtime]] for the action-space breakdown and [[rl-training-harness]] for the full architecture (both built on Stable-Baselines3's `ActorCriticPolicy`), plus [[isaac-sim-integration]] for the Isaac Sim training environment this policy also runs in.

## PolicyBridge — gating a second, focused policy

Bridges the always-on `TransformerPolicy`/`GodTransformerPolicy` with a separate continual-learning policy net (`cl_policy_net`, owned by `ContinualLearner`). Only active during "learning mode" — gated by the cognitive loop's N=5 sustained curiosity+surprise streak (see [[cognitive-loop]]). `predict_action(obs, task_label, deterministic)` routes through the focused net when learning mode is on; `decide()` in [[agent-runtime]] checks `is_in_learning_mode()` first and falls back to the direct policy otherwise. This was a real fixed bug too — `predict_action()` was built but never actually called from anywhere, so the whole learning-mode mechanism existed but never ran a real decision through it.

## SelfSupervisedTrainer — learning from surprise

Constructed with `world_model`, `brain`, `emotion_system`. `train_step(buffer)` returns `mean_surprise`, which the cognitive loop stores as `agent._last_sst_surprise` — this is one of the two inputs (with curiosity) to the N=5 learning-mode streak counter. Resolves the world model lazily from `brain.world_model` on every call rather than caching a reference at construction, specifically so attachment order relative to world-model init doesn't matter.

## SkillTracker — self-referential progress

`record_attempt(task_label, obs_vector, action_taken, imagined_target)` compares the action actually taken (one-hot encoded via the same `ACTION_TYPE_INDEX` scheme deliberation itself uses) against the most recent deliberation's own top choice. Returns `{'improving': bool, ...}`. There is no external teacher anywhere in this — an agent is only ever measured against its own recent past self. `get_weakest_task()` feeds Path 1 of the cognitive loop's focus-task resolution (see [[cognitive-loop]]).

## ContinualLearner — Avalanche-based, task-aware

Attached via `add_continual_learning(agent, strategy='replay')`. Owns an `experience_buffer` and its own `policy_net` (the one `PolicyBridge` wraps). `learn_from_buffer()` consumes-and-clears the buffer each cycle — a real fixed bug existed here: the plan's original wiring read the buffer _after_ calling `learn_from_buffer()`, by which point it was always empty, so `SelfSupervisedTrainer` silently trained on nothing every cycle. Fixed by snapshotting the buffer before calling `learn_from_buffer()`. `switch_task(task_id: int)` tells Avalanche a task boundary occurred (so replay/EWC strategies manage per-task buffers correctly instead of treating everything as one undifferentiated stream) — `task_id` is derived via a stable MD5 hash of the focus-task label, not Python's built-in `hash()`, specifically because `hash()` is randomized per-process since Python 3.3 and would make a task look "new" after every restart.

## GRPO — the actual policy update

`grpo_update(scored_actions, obs, lr=1e-4)` — monkeypatched onto the policy instance (not added to the `TransformerPolicy`/`GodTransformerPolicy` class definitions themselves) as a bound method. Runs from the most recent deliberation's `all_scored_actions`, returns `mean_advantage`, which the cognitive loop feeds into the per-domain reward history used for specialization detection (see [[cognitive-loop]]).

## God abilities

`GodControlSystem(god_type)` (`ai_core/god_controls.py`) exposes `.abilities` and `.ability_names()`; `integrate_god_controls(agent)` wires it up. The 5 extra action dims on `GodTransformerPolicy` (trigger_flag, ability_idx, param1-3) select and parameterize one of these per step — see [[agent-runtime]] for the encoding and [[god-abilities]] for the full per-god ability tables, two fixed indexing bugs, and how `use_god_ability()` actually routes outcomes back through the reward system.