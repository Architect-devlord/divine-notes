---
type: system
status: ingested
---
# RL Training Harness

💡 **What this is**: `rl/policy.py`, `rl/env.py`, `rl/train.py` — the Stable-Baselines3 PPO harness that trains [[agent-runtime]]'s policy networks offline, as opposed to the always-on learning [[reward-and-learning-stack]] describes happening during normal operation. Confirms the exact 13/18-dim action layout referenced everywhere else on this wiki, from the actual policy source rather than from callers' usage of it.

## `TransformerEncoder` — shared, and deliberately small

A small transformer (`d_model=128, nhead=4, num_layers=2`) shared by both policy classes — much smaller than [[world-model]]'s `d_model=512, n_layers=6`. Makes sense: this is a policy head reading an already-built 128-dim observation, not a multimodal world model building its own representations from raw vision/audio/proprioception.

## `TransformerPolicy` (13-dim) and `GodTransformerPolicy` (18-dim)

Both subclass Stable-Baselines3's `ActorCriticPolicy` directly — this is real PPO, not a custom RL loop. Confirms, at the source, the exact same 13-dim base layout documented in [[agent-runtime]] and [[god-abilities]]: `move_forward, move_strafe, jump, sneak, attack, use, drop, open_inv, swap_hand, yaw_delta, pitch_delta, sprint, hotbar_slot`.

`GodTransformerPolicy` adds a separate `GodAbilityHead` module (its own trunk, not sharing weights with the movement head) producing three sub-outputs: a trigger logit (sigmoid → Bernoulli sample), ability logits (softmax → Categorical sample over `n_abilities`), and 3 tanh-bounded params. Two separate `log_std` parameters exist — `log_std_base` (13-dim, movement noise) and `log_std_params` (3-dim, ability-targeting noise) — a deliberate choice so movement exploration and ability-targeting exploration can have independently-tuned noise scales rather than sharing one.

`decode()`'s docstring states the RL-relevant framing directly: because trigger is a learned Bernoulli rather than an always-on flag, **the policy can learn when _not_ to use an ability** — saving cooldowns, avoiding telegraphing a move — as naturally as learning when to use one.

## `DivineWorldEnv` — the Gymnasium wrapper

Wraps an `NPCAgent` instance as a standard `gym.Env`. Two fixed bugs worth knowing about if this harness is ever extended:

- Observation space was hardcoded `Box(shape=(50,), low=-10, high=10)` — a leftover from before [[observation-space]]'s 128-dim builder existed. Now imports `OBS_DIM` directly from `obs_builder.py` and tightens bounds to `[-1, 1]` to match what that module actually produces.
- Action space was `Box(shape=(11,))`, silently missing `sprint` and `hotbar_slot` (dims 11–12) — fixed to the correct 13.
- `step()` originally called a method (`update_emotion_memory()`) that doesn't exist on `NPCAgent` at all — the correct pairing is `agent.learn(...)` for the RL update plus a separate `agent.emotion.decay()` call for mood decay, and that's what it does now.

## `train.py` — standard PPO, run as a script

`train_agent()` builds `n_envs` parallel `DivineWorldEnv`s (via `SubprocVecEnv` if >1) around fresh `NPCAgent` instances, trains with SB3's `PPO` + the custom `TransformerPolicy`, with `CheckpointCallback`/`EvalCallback` and TensorBoard logging. One fixed environment quirk worth knowing: running `python rl/train.py` directly from the project root fails on the bare `from env import` / `from policy import` lines unless `rl/`'s own directory is added to `sys.path` first (it now is, explicitly, at the top of the file) — a classic "works when imported as a package, breaks when run as a script" issue.

## How this relates to the always-on learning in [[reward-and-learning-stack]]

Worth being clear about the relationship rather than assuming they're the same thing: this harness is **offline batch training** — spin up N parallel simulated agents, run PPO for however many timesteps, save a checkpoint. [[reward-and-learning-stack]]'s `SelfSupervisedTrainer`/`ContinualLearner`/GRPO path is what runs **during a live agent's normal operation**, learning continuously from whatever it actually experiences. Both ultimately produce weights for the same policy classes documented here — they're two different training regimes for the same model architecture, not competing systems.