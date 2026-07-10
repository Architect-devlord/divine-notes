---
type: file
status: ingested
---
# world_model.py

💡 **Role**: two systems in one file — the real neural "imagination" engine [[brain-core]]/[[imagination-and-planning]] call into, and (unrelated, confirmed) the Mental Matrix physics-sandbox backend. [[world-model]] already covers both in real depth — architecture, `EnsembleWorldModel`'s uncertainty mechanism, `imagine()`'s rollout, storage/training, and the full Mental Matrix class breakdown. This page is the mechanical catalog plus two genuinely new findings the topic page doesn't have: two places still using the pre-fix `action_dim=11` instead of the corrected 13, and `imagine()`'s self-acknowledged approximation in how it feeds predicted state back into the next step.

## Imports

`sys`, `pathlib.Path`, `torch`, `torch.nn`, `torch.nn.functional`, `torch.distributions` (`Normal`, `Categorical`), `numpy`, `typing`, `dataclasses`, `collections.deque`, `logging`, `datetime`, `asyncio`, `json` — all external/stdlib. One internal import: `from ai_core.config_loader import get_section, get_device`.

## Classes

**Mental Matrix classes** (`Vector3`, `PhysicsBody`, `SimulatedObject`, `MentalMatrixSimulation`, `MentalMatrixService`, `MentalMatrixWebSocketManager`, `MentalMatrixAgentClient`) — see [[world-model]]'s own dedicated section for the full method-by-method breakdown; not re-catalogued here to avoid duplicating a page this same session already wrote in detail.

**Neural world model classes:**

- `WorldModelConfig` (dataclass) — every architecture/training constant in one place; `action_dim: int = 13` carries the fix comment directly (`# FIX INT-05: was 11, must match TransformerPolicy.BASE_DIM=13`) — see "Not yet reconciled" below for two places that don't actually follow it.
- `VisionEncoder`, `AudioEncoder`, `ProprioceptionEncoder`, `ActionEncoder` (all `nn.Module`) — one CNN or Multi-Layer Perceptron (MLP) each, all with a single `forward()`; `VisionEncoder`/`AudioEncoder` both handle an optional leading time dimension by flattening batch×time before their conv stack and unflattening after.
- `TransformerBlock` (`nn.Module`) — one `forward()`: pre-norm multi-head attention + FFN, standard.
- `WorldModelTransformer` (`nn.Module`) — owns the four encoders above plus a learned positional embedding and the stack of `TransformerBlock`s; `forward()` sums all present-modality encodings into one representation before the transformer stack, rather than concatenating — worth knowing since it means every modality must already share `d_model` width going in.
- `VariationalEncoder` (`nn.Module`) — one `forward()`: the reparameterization trick, mean/logvar/sample.
- `RewardPredictor`, `TerminationPredictor`, `NextStatePredictor` (all `nn.Module`) — one `forward()` each, plain Multi-Layer Perceptrons (MLPs).
- `WorldModel` (`nn.Module`) — the assembled whole. `encode_observation()`, `forward()`, `imagine()`, `compute_loss()`, `train_step()`, `save()`/`load()`, `get_stats()` — see [[world-model]] for `imagine()`'s rollout logic and the ensemble's uncertainty mechanism; not repeated here. Two details not on that page: `compute_loss()` deliberately keeps `next_state_loss`/`kl_loss` as zero-valued tensors (not Python floats) when their inputs aren't present, specifically so the loss sum stays inside the autograd graph with a consistent dtype rather than breaking gradient flow; `forward()`'s next-state head is fed the _raw_, un-re-encoded action tensor concatenated onto the latent, deliberately, since the action was already folded into the latent once via `encode_observation()` and re-encoding it a second time would double-count it.
- `EnsembleWorldModel` — see [[world-model]]; not repeated here.

No async methods on any neural-network class — only the Mental Matrix side of this file uses `asyncio`.

## Functions

- `create_default_world_model(device)` — one-line convenience constructor.
- `test_world_model()` — a smoke test with dummy tensors. **Not yet reconciled with the `action_dim` fix**: its dummy `action` batch is `torch.randn(B, T, 11)` — the pre-fix dimension. If this test is ever actually run against the current `WorldModelConfig` default of 13, it will shape-mismatch inside `encode_observation()`.
- `integrate_world_model_with_agent(agent)` — constructs a `WorldModel`+buffer+trainer triple and monkeypatches `agent.brain.evaluate_event` to a neural version that falls back to the original method on any exception. See [[world-model]] as the single call site wiring all three together.
- `_build_observation_from_context(agent, context)` — see [[world-model]] for its role; one detail not on that page: **the same stale `11` reappears here** — the fallback branch when `agent.last_action` is `None` is `torch.zeros(1, 1, 11, ...)`, not 13. Both this and `test_world_model()`'s dummy batch look like the same fix landing in `WorldModelConfig` without every call site that constructs an action tensor by hand being updated to match — worth a grep for `action_dim` / hardcoded action-tensor widths across this file the next time it's touched, in case there are more.
- `get_mental_matrix_router()` / `register_mental_matrix_api(app)` — see [[world-model]].

## Problems (faced by traditional AI systems / LLMs)

World-model-based imagination has a specific, well-known failure mode distinct from the ensemble-uncertainty problem [[world-model]] already covers: a single deterministic rollout tells a planner nothing about how _reliable_ that rollout is, so a planner using it alone can't distinguish a confident correct prediction from a confident wrong one. Separately, more mundane but just as real: any codebase where a fixed-width tensor shape gets corrected in one canonical place (a config default) but constructed by hand in several others is one dimension-mismatch bug away from silently failing the moment a rarely-exercised code path (a fallback, a smoke test) actually runs.

## Solutions

The ensemble (see [[world-model]]) is the answer to the first problem. The second doesn't really have a "solution" in this file yet — it's an open finding, not a fix: `WorldModelConfig.action_dim` is the single source of truth the config-driven paths already respect, but `test_world_model()` and `_build_observation_from_context()`'s fallback both construct an 11-wide tensor by literal value instead of reading `config.action_dim`, so the correction three call sites away (`TransformerPolicy.BASE_DIM=13`) didn't propagate here.

## Files Required

- `ai_core/config_loader.py` — `get_section()`, `get_device()`. Not yet given its own file-level page.

## Files Used In

- `brain_core.py` / `planner.py` — both lazily import `_build_observation_from_context()`; `brain_core.py` also calls `set_world_model()`/`wm.imagine()` (see [[brain_core]] and [[planner]]).
- `self_supervised_trainer.py` — `WorldModel.train_step()` is the actual call `SelfSupervisedTrainer.train_step()` delegates to (see [[self_supervised_trainer]]).
- `agent.py` — constructs the `WorldModel`/`EnsembleWorldModel` per the `use_ensemble` config flag (see [[agent]] and [[agent-runtime]]).
- [[electron-frontend]] — explicitly _not_ used by the Mental Matrix side of this file, despite the shared name; see [[world-model]]'s correction note.