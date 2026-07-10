---
type: file
status: ingested
---
# env.py (`rl/env.py`)

💡 **Role**: a Gymnasium-compatible training environment — but a simulated one, not a live connection to Minecraft. Distinct from the real runtime path (`communication_protocol.py`'s WebSocket handlers): this is for training/testing a policy against synthetic outcomes, not for actual gameplay.

## Imports

`gymnasium` (`Env`, `spaces`), `numpy`, `typing` — external. No internal project imports — sizes itself against fixed constants rather than importing `obs_builder.OBS_DIM`/`policy.BASE_DIM` directly (see "Not yet reconciled" below).

## Classes

### `DivineWorldEnv` (extends `gymnasium.Env`)

**Not yet reconciled**: the class's own docstring states `Observation: Box(50,)` / `Action: Box(11,)` — the pre-fix dimensions, matching the same stale-11 pattern already found three times elsewhere in this ingest ([[wiki/codebase/files/world_model|world_model.py]] twice, [[wiki/codebase/files/vision|vision.py]] once). Here it's the _docstring_ that's stale rather than a hardcoded tensor: `__init__` actually sets `observation_space = Box(low=-1, high=1, shape=(128,))` and `action_space = Box(low=-1, high=1, shape=(13,))` — the corrected dimensions — so the class works correctly, but the class-level documentation contradicts its own constructor. A fourth instance of the same underlying pattern (a fix landing in the code without every place that describes the old shape being updated to match), just one step removed from an actual bug this time.

**Methods:**

- `__init__(render_mode=None)` — sets up the (correctly-sized) observation/action spaces, plus internal step counters and a max-episode-length cap.
- `reset(seed, options)` — returns a zeroed/randomized initial observation; no connection to any real Minecraft state.
- `step(action)` — the core of this being a _simulated_ environment: `_simulate_outcome(action)` produces a synthetic next-observation and reward from simple heuristics (not a real physics or game-state transition), rather than sending the action anywhere real and reading back what actually happened.
- `_simulate_outcome(action)` — the synthetic-outcome generator; a placeholder good enough to let a policy be trained/tested against _something_ shaped like the real observation/action spaces, without needing a live Minecraft connection or [[world-model]]'s neural world model running.
- `render()` — a no-op or minimal print, depending on `render_mode`; no real visualization.
- `close()` — standard Gymnasium cleanup hook.

No async methods, no module-level functions.

## Problems (faced by traditional AI systems / LLMs)

Training or testing a policy against a real, live game connection is slow and requires the whole stack (Minecraft, the mod, the WebSocket bridge) to be running — a real friction point for quick iteration or CI. A narrower, more mundane problem: any class docstring that describes shapes/behavior at the time it was written will drift out of sync with the actual implementation the moment those shapes change, unless something forces the two to be checked together.

## Solutions

A synthetic, Gymnasium-standard environment lets a policy's basic training loop be exercised without any of the real infrastructure — at the cost of the "outcomes" not reflecting real game dynamics at all, which is an accepted tradeoff for what this environment is for (structural testing, not behavior training). The docstring drift doesn't have a solution in this file yet — it's an open finding, like the similar cases elsewhere in this ingest: the fix should either update the docstring to match `Box(128,)`/`Box(13,)`, or better, have both the docstring and the constructor read from `obs_builder.OBS_DIM`/`policy.BASE_DIM` directly so they can't diverge again.

## Files Required

None imported directly — sizes are hardcoded rather than read from [[wiki/codebase/files/obs_builder|obs_builder.py]]'s `OBS_DIM` or [[wiki/codebase/files/policy|policy.py]]'s `BASE_DIM`, which is itself part of why the docstring and the constructor could drift independently.

## Files Used In

- Presumably a training script that constructs this environment and drives a [[wiki/codebase/files/policy|policy.py]] policy against it — not directly traced to a specific call site this pass.