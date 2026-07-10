---
type: file
status: ingested
---
# policy_bridge.py

💡 **Role**: shares one encoder between the always-on `TransformerPolicy` and a small continual-learning head, so the two representations never drift apart. [[reward-and-learning-stack]] already covers the concept (learning-mode gating, `predict_action()`'s "never called" bug and its fix) — this page is the actual encoder-finding and soft-sync mechanics behind that, neither of which is on the topic page.

## Imports

`torch`, `torch.nn`, `numpy`, `typing`, `logging` — all external. No internal project imports; `transformer_policy`/`cl_policy_net` arrive pre-built through the constructor, this file never imports their classes.

## Classes

### `PolicyBridge` (`nn.Module`)

**Class constant**: `SYNC_EVERY = 200` — soft-sync cadence, in forward calls.

**Methods:**

- `__init__(transformer_policy, cl_policy_net, obs_dim, action_dim)` — probes for a usable encoder once (see `_find_encoder()` below) and builds `cl_head` (a 2-layer Multi-Layer Perceptron (MLP)) sized to whatever that probe returns.
- `_find_encoder(policy, obs_dim)` — **the fix this file exists for.** The original version hardcoded `transformer_policy.policy.mlp_extractor.policy_net` — wrong on two counts: `transformer_policy` _is_ the policy object itself (no `.policy` sub-attribute to descend into), and `TransformerPolicy` overrides `_build_mlp_extractor()` to set `self.features_extractor` instead of ever building SB3's default `MlpExtractor`. Both assumptions being wrong meant every call silently fell through to a 128-dim fallback, and `cl_head` trained on raw, unencoded observations instead of the policy's real internal representation — the continual-learning head was learning from a completely different input than the one the actual policy uses to decide anything. Now tries four candidate paths in order, verifying each one actually produces output on a dummy tensor before trusting it (having the attribute isn't enough on its own).
- `_encode(obs_tensor)` — runs the found encoder under `no_grad()`; passthrough (returns the input unchanged) if no encoder was found or the call raises.
- `predict_action(obs, task_label, deterministic)` — the main entry point. Routes to `cl_head` only if learning mode is active _and_ `task_label` matches the currently active task; otherwise calls `transformer_policy.predict()` directly. Triggers `_soft_sync_cl_head()` every `SYNC_EVERY` calls as a side effect.
- `set_learning_mode(active, task_label)` — the only way learning mode changes; called exclusively by [[cognitive_loop]], never forced from elsewhere.
- `get_cl_head_params()` — `cl_head`'s parameters, for Avalanche's optimizer.
- `is_in_learning_mode()` / `get_active_task()` — plain state reads; `agent.py`'s `decide()` checks the former before deciding whether to route through this bridge at all (see [[agent]]).
- `_soft_sync_cl_head(tau=0.05)` — **a second fix in the same file.** The original version indexed `transformer_policy.policy.mlp_extractor.policy_net[-1]`, wrong for the same reason `_find_encoder()` was, plus a second, independent problem even once pointed at the real encoder: `TransformerEncoder` isn't an `nn.Sequential`, so `[-1]` indexing isn't valid on it at all, and its actual last operation is a `LayerNorm` — a per-channel scale, not a weight matrix there's anything meaningful to copy. Now searches the encoder's named modules for the last `nn.Linear` whose output size matches `cl_head`'s input size, and soft-copies 5% of that layer's weights in per call; skips the sync entirely (rather than guessing) if no matching layer is found.

No async methods — every operation here is a synchronous forward pass or weight copy.

## Problems (faced by traditional AI systems / LLMs)

Any system built from two networks that are supposed to share a representation — here, a fast always-on policy and a slower continual-learning head — silently stops sharing anything the moment the "shared" path is wrong, and nothing about the training loop signals this: losses still compute, gradients still flow, weights still update. The failure is invisible from the training metrics alone; it only shows up as the second network learning something subtly (or not-so-subtly) different from what was intended.

## Solutions

Verify, don't assume: `_find_encoder()` doesn't trust that an attribute path exists just because the architecture diagram says it should — it runs a dummy tensor through each candidate and only accepts one that actually produces output of a sane shape. `_soft_sync_cl_head()` applies the same discipline to weight-copying: search for a structurally compatible layer by its actual shape, rather than assuming a fixed index into an architecture that turned out not to have the sequential structure the original code assumed.

## Files Required

None from within this codebase — `transformer_policy` and `cl_policy_net` are passed in fully constructed; this file never imports `rl/policy.py` or `continual_learner.py` directly.

## Files Used In

- `agent.py` — constructs this after both `policy` and `continual_learner` exist, and `decide()` checks `is_in_learning_mode()` before routing through it (see [[agent]] and [[agent-runtime]]).
- `cognitive_loop.py` — the only caller of `set_learning_mode()`, gated on the N=5 curiosity+surprise streak (see [[cognitive_loop]]).
- `continual_learner.py` — `get_cl_head_params()` feeds this bridge's `cl_head` into Avalanche's own optimizer (see [[continual_learner]]).