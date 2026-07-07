---
type: system
status: ingested
---
# World Model

💡 **What this is**: `ai_core/world_model.py` — the actual neural network [[brain-core]] and [[imagination-and-planning]] call "imagination." Also, unexpectedly, where the Electron frontend's Mental Matrix visualizer backend lives (see below) — the same file does two fairly different jobs.

## Architecture

`WorldModelConfig`: `d_model=512, n_heads=8, n_layers=6, d_ff=2048, dropout=0.1`. Four per-modality encoders (`VisionEncoder`, `AudioEncoder`, `ProprioceptionEncoder` [32-dim: health/hunger/position/etc.], `ActionEncoder`) feed into a stack of 6 `TransformerBlock`s (standard pre-norm MHA + FFN). `action_dim=13` by default — a fixed bug: this used to be 11, silently mismatched against `TransformerPolicy.BASE_DIM=13` (see [[agent-runtime]]).

**VAE option** (`use_vae=True` by default, `latent_dim=256, kl_weight=0.1`): a `VariationalEncoder` sits between the transformer and the prediction heads, giving the model a genuine distributional latent rather than a point estimate — this is _separate_ from the ensemble's uncertainty estimate (below); VAE uncertainty is about the representation itself, ensemble uncertainty is about model disagreement.

**Three prediction heads**: `RewardPredictor`, `TerminationPredictor` (death/failure probability), `NextStatePredictor`. All three feed `WorldModel.forward()`'s output.

## `EnsembleWorldModel` — where the uncertainty penalty actually comes from

Five independent `WorldModel` instances (`n_models=5`), each trained the same way. `forward()` runs all five and returns the **mean and std** across `reward`, `termination`, and `next_state`. This is exactly the `next_state_std` value [[brain-core]]'s deliberation reads to compute its uncertainty penalty — confirmed at the source: `ensemble.forward()` really does return `next_state_std`, not an approximation of it.

## `imagine()` — the actual rollout

Autoregressive, not a single forward pass: at each of `steps` timesteps, it takes the _previous_ imagined state, feeds it back in as the new context (along with the next planned action), and calls `forward()` again — the same Dreamer/latent-imagination pattern the file's own header credits as inspiration (alongside GATO, World Models, Decision Transformer). Returns a trajectory dict of `rewards`, `terminations`, `states` — exactly what [[imagination-and-planning]]'s `ImagineResult` wraps.

## Storage and training

`WorldModelReplayBuffer(capacity=100_000, sequence_length=64)` — larger than earlier documentation on this wiki assumed (an earlier pass on [[reward-and-learning-stack]] stated 50,000; 100,000 is the actual default parameter, confirmed at the source — worth reconciling if that page is revisited). `WorldModelTrainer(world_model, replay_buffer, batch_size=16)` runs the actual training step and tracks stats for save/resume. `integrate_world_model_with_agent(agent)` is the single call site that wires all three together onto a fresh `NPCAgent`.

`_build_observation_from_context(agent, context)` — worth knowing this exists here specifically: it's the function that converts [[brain-core]]'s lightweight `{health, hunger, emotions, novelty, urgency}` context dict into the full tensor dict the transformer actually expects. Both `brain_core.py` and `planner.py` import it from here rather than duplicating the conversion.

**Not yet reconciled**: `WorldModelConfig.action_dim` was fixed from 11 to 13 (matching `TransformerPolicy.BASE_DIM`), but two hand-built tensor constructions elsewhere in this same file never got updated to match — `test_world_model()`'s dummy batch and `_build_observation_from_context()`'s no-last-action fallback both still hardcode a width-11 action tensor. Neither is provably broken today (the fallback only fires when `agent.last_action` is `None`, and the test function may not be exercised in normal operation), but both would shape-mismatch against the rest of the pipeline if actually triggered. See [[wiki/codebase/files/world_model|world_model.py]]'s file-level page for exactly where.

## Unexpected discovery: this file also hosts the Mental Matrix backend

Roughly the first third of `world_model.py` (before the neural-network classes even start) is a separate system: `Vector3`, `PhysicsBody`, `SimulatedObject`, `MentalMatrixSimulation`, `MentalMatrixService`, `MentalMatrixWebSocketManager`, `MentalMatrixAgentClient`, plus `register_mental_matrix_api(app)`. This also resolves where `mental_matrix_api.py` (mentioned in earlier ingest passes as a separate file) actually connects — it's registered from inside this same module via `register_mental_matrix_api()`, not a fully separate subsystem.

**Correction, 2026-07-05**: the previous version of this section guessed that this backend "looks like it's used to give the Electron frontend's `WorldModelVisualizer.jsx`/`MentalMatrixSimulator.jsx` something visual to render." **Confirmed by Devlord, and independently by reading both sides, that this is wrong** — they're not connected. `MentalMatrixSimulator.jsx` (see [[electron-frontend]]) is a purely local, client-side Three.js physics toy with no fetch or WebSocket call to any backend anywhere in the file. The name and premise ("a sandbox you can watch objects move in") match closely enough that it's an easy pair to assume are linked — they aren't, at least not currently. Recording the correction rather than quietly editing over it, per this vault's usual rule.

## Mental Matrix classes, now read in full (lines 99–810)

Four dataclasses (`Vector3`, `PhysicsBody`, `SimulatedObject` — each with a matching `to_dict()`/`from_dict()` pair for JSON round-tripping) back a genuinely simple, hardcoded physics sim: `MentalMatrixSimulation` steps position by velocity, applies a constant `GRAVITY = 9.81` and per-object `friction`/`elasticity`, and bounces objects off a flat `GROUND_LEVEL = 0.0` — plain Euler integration, nothing learned. **Despite the class's own docstring calling it "powered by world model predictions," nothing in this section calls `WorldModel`, `EnsembleWorldModel`, `forward()`, or `imagine()` anywhere** — that phrase describes an intent this code doesn't currently carry out.

`MentalMatrixService` is the per-agent registry (`get_or_create_simulation`) plus a single global 60Hz loop (`start_update_loop`) that steps every registered agent's simulation each tick, and file export/import. `MentalMatrixWebSocketManager` fans a simulation's events out to however many browser clients are subscribed to that `agent_id`, and accepts a full command set over the socket: `add_object`, `remove_object`, `apply_impulse`, `set_running`, `reset`, `set_time_scale`, `update_object`, `export`, `import`. `MentalMatrixAgentClient` is the mirror image — a client class an **agent itself** could use to connect and script scenarios (`simulate_scenario()`, `test_physics_interaction()`), reset, add objects, run for N seconds, then export the result. All of this is registered onto FastAPI via `register_mental_matrix_api(app)` — which actually lives at line 2005, near the very end of the file, not inside the 99–810 range as an earlier pass on this page assumed — mounting an `APIRouter(prefix="/mental-matrix")` with a `/ws` WebSocket endpoint, a `/health` check, and an HTTP control endpoint mirroring a subset of the WS commands. `py_backend/mental_matrix_api.py` is confirmed as a pure re-export shim (its own docstring says so): all logic lives here in `world_model.py`; the shim exists purely so `main.py` can `include_router()` without depending on the new location directly, and it's listed in `Config.AGENT_EXCLUDE_MODULES` since it's server-only and never bundled into packaged agent executables.

**Why this matters for the frontend-disconnection finding above**: this backend's command set (`add_object`/`remove_object`/`apply_impulse`/`set_running`/`reset`/`set_time_scale`) is a near-exact conceptual match for what `MentalMatrixSimulator.jsx` independently reimplements client-side in Three.js — add primitives, drag/impulse, play/pause, reset, a speed slider. That's a strong signal the two were meant to be the same feature. They aren't currently wired together (confirmed above), but if that ever gets fixed, this WebSocket protocol is almost certainly the intended integration point rather than something needing to be designed from scratch.