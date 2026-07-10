---
type: file
status: ingested
---
# policy.py (`rl/policy.py`)

💡 **Role**: the two Stable-Baselines3-derived policy classes every "policy" reference throughout this ingest (`agent.py`'s `decide()`, `policy_bridge.py`, `self_supervised_trainer.py`'s `grpo_update()`) actually points to. Not previously read directly — only inferred from how other files describe using it.

## Imports

`torch`, `torch.nn`, `numpy`, `typing`, `gymnasium.spaces`, `stable_baselines3.common.policies.ActorCriticPolicy` — external. No internal project imports; this file is a pure SB3 extension, deliberately isolated from the rest of `ai_core`.

## Classes

### `TransformerPolicy` (extends `ActorCriticPolicy`)

**Class constant**: `BASE_DIM = 13` — the NPC action-vector width [[wiki/codebase/files/agent|agent.py]]'s `act()` and [[wiki/codebase/files/world_model|world_model.py]]'s `WorldModelConfig.action_dim` both fix corrections against.

**Methods:**

- `_build_mlp_extractor()` — overridden to build `self.features_extractor` (a small transformer encoder over the 128-dim observation) instead of SB3's default `MlpExtractor` — the exact override [[policy_bridge]]'s `_find_encoder()` had to learn to detect rather than assume, since it isn't the standard SB3 shape.
- `forward(obs, deterministic)` — encodes via the transformer, outputs an action mean directly (`Tanh`-bounded to `[-1, 1]`) plus a learned `log_std` parameter (not obs-dependent — one global std per action dimension, not a per-state one).
- `_predict(obs, deterministic)` — samples from `Normal(mean, exp(log_std))` if not deterministic, else returns the mean directly; this is the actual call [[agent]]'s `decide()` makes when not routed through [[policy_bridge]].
- `predict(obs, state, deterministic)` — the standard SB3-compatible wrapper (`(action, state)` tuple return) some callers expect over the raw `_predict()`.

### `GodTransformerPolicy` (extends `TransformerPolicy`)

**Class constant**: `TOTAL_DIM = 18` — `BASE_DIM` (13, inherited: movement + look + flags + hotbar) plus 5 more: a continuous ability-intensity value, two discrete logits for "which ability" (`log_std_params`, separate from the inherited `log_std_base` covering only the first 13 dims), and two spare/reserved dims. This is the exact width [[wiki/codebase/files/agent|agent.py]]'s `act_god()` reads from and the same figure `communication_protocol.py`'s B-15 fix checks (`len(action_array) >= 18`).

**Methods:**

- `forward(obs, deterministic)` — extends the base `forward()`: the continuous dims come from the same transformer output as `TransformerPolicy`; the two discrete ability logits go through their own small linear head and a `Categorical` distribution, concatenated onto the continuous output rather than replacing any of it.
- `_predict(obs, deterministic)` — samples continuous dims from `Normal` (using the concatenated `log_std_base`+`log_std_params`, zero-padded for the two reserved dims — the exact construction [[self_supervised_trainer]]'s `grpo_update()` fix had to reproduce manually) and the two discrete dims from `Categorical`, concatenated into one 18-dim array.

No async methods, no module-level functions — this file is purely the two class definitions.

## Problems (faced by traditional AI systems / LLMs)

Extending a well-established RL library's policy class (SB3's `ActorCriticPolicy`) with a genuinely different internal architecture (a transformer instead of the default MLP extractor, a mixed continuous+discrete action head for gods) creates a policy that's still nominally SB3-compatible at the interface level but silently diverges from SB3's own internal assumptions about _how_ that interface is implemented — exactly the gap that produced both [[policy_bridge]]'s wrong-attribute-path bug and [[self_supervised_trainer]]'s `evaluate_actions()` bug, in two completely different files, from the same underlying cause.

## Solutions

Not really solved _in this file_ — the fixes for both problems above live in the files that were calling into this one incorrectly, not here. This file's own solution is narrower: `GodTransformerPolicy` extending `TransformerPolicy` (rather than being a separate parallel class) means the shared continuous-action machinery — encoder, base 13 dims, `Normal` sampling — is written once and inherited, with only the ability-specific discrete head genuinely new code.

## Files Required

None from within this codebase — pure SB3 extension.

## Files Used In

- `agent.py` — `initialize_policy()` picks between the two classes by `god_type` (see [[agent]] and [[agent-runtime]]).
- [[policy_bridge]] — wraps whichever instance is constructed; `_find_encoder()`/`_soft_sync_cl_head()` both had to reverse-engineer this file's actual structure rather than assuming SB3 defaults.
- [[self_supervised_trainer]] — `grpo_update()` is monkeypatched onto instances of these classes, and its `log_std`/`log_std_base`+`log_std_params` handling directly mirrors the two classes' own distinct distribution-construction logic.
- `rl/env.py` — `DivineWorldEnv`'s action/observation spaces are sized to match these classes' expectations (see [[wiki/codebase/files/env|env.py]]).