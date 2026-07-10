---
type: file
status: ingested
---

# continual_learner.py

💡 **Role**: the Avalanche-based lifelong-learning wrapper — trains a policy on accumulated experience without catastrophically forgetting older tasks, and (as of commit `8839776`) also owns the [[emergent_templates|EmergentSkillPool]] that feeds newly-discovered skills into the live planner. [[reward-and-learning-stack]] already covers this file's buffer-consumption bug fix and its MD5-based task-ID derivation — not repeated here.

## Imports

`torch`/`torch.nn`, `numpy`, `collections.deque`, `typing`, `pathlib.Path`, `logging` — all external. `avalanche` (several submodules) is imported inside a `try`/`except ImportError`, gating the whole file's usability behind `AVALANCHE_AVAILABLE`. One internal, lazy import, inside `ContinualLearner.__init__` rather than at module level: `from ai_core.emergent_templates import EmergentSkillPool` — see [[emergent_templates]].

## Classes

### `PolicyNetwork`
A plain 4-layer Multi-Layer Perceptron (MLP) (`nn.Module`): `obs_dim → hidden → hidden → hidden/2 → action_dim`, ReLU + LayerNorm + Dropout(0.1) on the first two blocks, `Tanh` output to keep actions in `[-1, 1]`. This is a separate, smaller network from the live `TransformerPolicy` — not a copy of it, which is precisely why `_sync_weights_from_live_policy()` below has to exist.

**Methods:**
- `forward(x)` — standard forward pass through the `nn.Sequential` stack.

### `ValueNetwork`
A smaller 3-layer Multi-Layer Perceptron (MLP) critic, same normalization pattern, single scalar output — no activation on the last layer since a value estimate isn't bounded.

**Methods:**
- `forward(x)` — forward pass.

### `ContinualLearner`
The class actually attached to an agent (`agent.continual_learner`). Owns both networks above, an Avalanche training strategy (`Naive`/`Replay`/`EWC`/`LwF`, chosen at construction), the `EmergentSkillPool`, and the raw `experience_buffer` list that `collect_experiences()` fills and `learn_from_buffer()` drains.

**Methods:**
- `__init__(agent, strategy, hidden_dim, memory_size)` — raises immediately if Avalanche isn't installed. Fixed dimensions worth knowing if either ever drifts again: `obs_dim = 128` (was 50 — must track `obs_builder.OBS_DIM`) and `action_dim = 13` (must track `TransformerPolicy.BASE_DIM`) are hardcoded here independently of wherever else those constants live, so a change to either upstream needs a matching manual update in this file too.
- `_init_avalanche_strategy()` — builds the Adam optimizer (separate LRs for policy vs. value net), the `EvaluationPlugin`, and picks one of the four Avalanche strategy classes by name. Also where the rolling MSE deque (`_mse_history`, `maxlen=100`) gets created, replacing a removed `accuracy_metrics()` call that didn't make sense for continuous action vectors.
- `collect_experiences(min_batch_size)` — turns `agent.brain.continual_buffer` entries into `(obs, action, reward, next_obs, done, task_id)` tuples. Obs priority: an event's own stored `obs` → `agent.last_obs` → a 4-value normalized context stub padded with zeros. Action priority: an event's own stored `action` array → a one-hot guess from `event['type']` against a hardcoded 11-entry template name list. Returns `False` (declining to train this cycle) if fewer than `min_batch_size` experiences are available.
- `_sync_weights_from_live_policy()` — not weight-copying (impossible; different architectures) but policy distillation: runs the live `TransformerPolicy` on the agent's most recent real observation to get a target action, then appends that `(obs, target_action, reward=0.0, ...)` pair to the experience buffer. The zero reward is deliberate — it's a distillation target, not a real game outcome — so this pulls `PolicyNetwork` toward mimicking the live policy's current behavior rather than only ever learning from past, possibly-stale experience.
- `learn_from_buffer(epochs)` — the main entry point, called externally (see Files Used In). Order matters: syncs distillation weights first, then feeds the skill pool (wrapped in its own `try`/`except` so a clustering failure can never block the actual training step below it), then converts the buffer to an Avalanche dataset and calls `self.strategy.train()`. Clears `experience_buffer` only after a successful run.
- `_create_avalanche_dataset()` — stacks the buffer's observations/actions/task-labels into tensors and wraps them in an `AvalancheTensorDataset`.
- `switch_task(new_task_id)` — records a task boundary for Avalanche's benefit and updates `agent.brain.current_task`.
- `predict_action(observation)` / `predict_value(observation)` — inference-only forward passes through the two networks (`eval()` + `no_grad()`), independent of the live `TransformerPolicy`.
- `save(path)` / `load(path)` — full state persistence (both network state dicts, task history, stats) via `torch.save`/`torch.load`. `load()` warns and no-ops if the path doesn't exist rather than raising.
- `get_stats()` — merges the running `self.stats` dict with current task/buffer info.
- `get_recent_mse()` / `get_avg_mse()` — read access to the rolling MSE window described under `_init_avalanche_strategy()` above.

No async methods — Avalanche's `strategy.train()` call is synchronous and blocking; nothing in this file awaits anything.

## Functions

- `add_continual_learning(agent, strategy, **kwargs)` — the module-level convenience wrapper every caller actually uses instead of instantiating `ContinualLearner` directly: no-ops with a warning if Avalanche isn't installed, otherwise constructs the learner and assigns it to `agent.continual_learner`.

## Problems (faced by traditional AI systems / LLMs)

Catastrophic forgetting is the standard failure mode for any network trained sequentially on shifting data distributions: gradient steps that help the current task actively overwrite weights an earlier task depended on, with no mechanism to notice the damage. A second, narrower problem this file addresses directly: a small auxiliary learner trained independently of a much larger live policy will simply drift away from what the agent is actually doing, unless something explicitly pulls it back toward the live policy's current behavior.

## Solutions

Avalanche's `Replay`/`EWC`/`LwF` strategies are the standard, off-the-shelf answers to the first problem — this file is mostly integration work around them (dimension-matching, a defensive metric replacement, real observation/action reconstruction instead of the near-constant stub an earlier version used). The drift problem gets a purpose-built answer instead: periodic distillation from the live `TransformerPolicy` into `PolicyNetwork`'s own training buffer, so a much smaller, separately-trained network still tracks what the agent is actually doing, not just what its own limited experience buffer happened to contain.

## Files Required

- [[emergent_templates]] — `EmergentSkillPool`, instantiated in `__init__` and fed inside `learn_from_buffer()`.

## Files Used In

- `agent.py` — this is where `add_continual_learning(self, strategy='replay')` actually gets called, and where `SkillTracker` is constructed from `self.continual_learner` (not yet given its own file-level page — see [[agent-runtime]] for the narrative treatment in the meantime).
- [[cognitive_loop]] — reads `agent.continual_learner` at several points to trigger `learn_from_buffer()`.
- [[policy_bridge]] — reads `continual_learner.policy_net` directly to build its gated `cl_head` (not yet given its own file-level page).
- [[skill_tracker]] — holds a `continual_learner` reference for its own progress tracking (not yet given its own file-level page).
