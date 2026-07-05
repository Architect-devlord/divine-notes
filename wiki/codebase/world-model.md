---
type: system
status: ingested
---

# World Model

💡 **What this is**: `ai_core/world_model.py` — the actual neural network [[brain-core]] and [[imagination-and-planning]] call "imagination." Also, unexpectedly, where the Electron frontend's Mental Matrix visualizer backend lives (see below) — the same file does two fairly different jobs.

## Architecture

`WorldModelConfig`: `d_model=512, n_heads=8, n_layers=6, d_ff=2048, dropout=0.1`. Four per-modality encoders (`VisionEncoder`, `AudioEncoder`, `ProprioceptionEncoder` [32-dim: health/hunger/position/etc.], `ActionEncoder`) feed into a stack of 6 `TransformerBlock`s (standard pre-norm MHA + FFN). `action_dim=13` by default — a fixed bug: this used to be 11, silently mismatched against `TransformerPolicy.BASE_DIM=13` (see [[agent-runtime]]).

**VAE option** (`use_vae=True` by default, `latent_dim=256, kl_weight=0.1`): a `VariationalEncoder` sits between the transformer and the prediction heads, giving the model a genuine distributional latent rather than a point estimate — this is *separate* from the ensemble's uncertainty estimate (below); VAE uncertainty is about the representation itself, ensemble uncertainty is about model disagreement.

**Three prediction heads**: `RewardPredictor`, `TerminationPredictor` (death/failure probability), `NextStatePredictor`. All three feed `WorldModel.forward()`'s output.

## `EnsembleWorldModel` — where the uncertainty penalty actually comes from

Five independent `WorldModel` instances (`n_models=5`), each trained the same way. `forward()` runs all five and returns the **mean and std** across `reward`, `termination`, and `next_state`. This is exactly the `next_state_std` value [[brain-core]]'s deliberation reads to compute its uncertainty penalty — confirmed at the source: `ensemble.forward()` really does return `next_state_std`, not an approximation of it.

## `imagine()` — the actual rollout

Autoregressive, not a single forward pass: at each of `steps` timesteps, it takes the *previous* imagined state, feeds it back in as the new context (along with the next planned action), and calls `forward()` again — the same Dreamer/latent-imagination pattern the file's own header credits as inspiration (alongside GATO, World Models, Decision Transformer). Returns a trajectory dict of `rewards`, `terminations`, `states` — exactly what [[imagination-and-planning]]'s `ImagineResult` wraps.

## Storage and training

`WorldModelReplayBuffer(capacity=100_000, sequence_length=64)` — larger than earlier documentation on this wiki assumed (an earlier pass on [[reward-and-learning-stack]] stated 50,000; 100,000 is the actual default parameter, confirmed at the source — worth reconciling if that page is revisited). `WorldModelTrainer(world_model, replay_buffer, batch_size=16)` runs the actual training step and tracks stats for save/resume. `integrate_world_model_with_agent(agent)` is the single call site that wires all three together onto a fresh `NPCAgent`.

`_build_observation_from_context(agent, context)` — worth knowing this exists here specifically: it's the function that converts [[brain-core]]'s lightweight `{health, hunger, emotions, novelty, urgency}` context dict into the full tensor dict the transformer actually expects. Both `brain_core.py` and `planner.py` import it from here rather than duplicating the conversion.

## Unexpected discovery: this file also hosts the Mental Matrix backend

Roughly the first third of `world_model.py` (before the neural-network classes even start) is a separate system: `Vector3`, `PhysicsBody`, `SimulatedObject`, `MentalMatrixSimulation`, `MentalMatrixService`, `MentalMatrixWebSocketManager`, `MentalMatrixAgentClient`, plus `register_mental_matrix_api(app)`. Based on naming alone (not yet deep-read): this looks like a separate, simplified physics sandbox used to give the Electron frontend's `WorldModelVisualizer.jsx`/`MentalMatrixSimulator.jsx` (see [[architecture-overview]] — still flagged as not-yet-ingested) something visual and human-watchable to render, distinct from the actual neural computation above. This also resolves where `mental_matrix_api.py` (mentioned in earlier ingest passes as a separate file) actually connects — it's registered from inside this same module via `register_mental_matrix_api()`, not a fully separate subsystem.

**Not yet deep-ingested**: the Mental Matrix classes themselves (lines ~99–810 of this file) weren't read in depth this pass — this note exists so a future session knows where to look rather than having to rediscover it.
