---
type: system
status: ingested
---
# Isaac Sim Integration

💡 **What this is**: `py_backend/isaac_sim_integration.py` — the actual bridge to Isaac Sim, and direct, concrete confirmation that the design wiki's Layer 2 ("Isaac Sim as training ground," see `Robots.md`) is a real, currently-maintained integration rather than an aspiration. Targets **Isaac Sim 5.0** specifically (NVIDIA open-sourced it at SIGGRAPH 2025) — the file's header documents a real migration from the 4.x API (`omni.isaac.core.articulations.ArticulationController` → `isaacsim.core.prims.SingleArticulation`, among several other renamed/restructured calls), so this has clearly been kept current against a real upstream API change, not written once and abandoned.

## Two classes

- **`IsaacSimActuator`** — translates an `NPCAgent`'s controls dict into Isaac joint commands via `ArticulationAction`, the 5.0-era API (`art.apply_action(ArticulationAction(joint_positions=positions))` rather than the old `apply_dof_position_targets`). **Open question, not resolved this pass**: [[perception-and-actuation]] documents a similarly-named `ActuatorAdapterIsaacSim` inside `actuators.py`, also doing Isaac Sim joint actuation via its own `_DOFProfile` abstraction. Whether these are the same bridge described from two angles, two genuinely different layers, or a real duplication worth flagging alongside [[known-issues]]'s other duplicate-implementation findings — wasn't confirmed. Worth a direct side-by-side read before assuming either.
- **`IsaacSimIntegration`** — manages one or more agents running inside a live Isaac Sim world.

## The folder-watcher — a real, working version of the design wiki's import/export idea

This is the most concrete finding in this file, and it directly validates a specific design-wiki claim rather than just gesturing at the general concept. `IsaacSimIntegration.__init__` spins up a background thread (`_poll_watch_folder`) that polls a `watch_folder` (default `dw_agents_sim/`) every `poll_interval_secs` (default 5.0) for new subdirectories. When one appears, `start_sim_agent(agent_id, entry)` fires automatically and looks for `brain.pcap` inside it — **the exact same BrainCapsule file format** [[memory-and-braincapsule]] documents as the portable identity object. If found, that capsule gets loaded straight into the running simulation.

In other words: drop an agent's BrainCapsule folder into `dw_agents_sim/`, and within 5 seconds it's a live agent inside Isaac Sim, no manual wiring required. This is a working, code-level instance of the design wiki's "Agent Import/Export System" concept (`Robots.md`) — for the Isaac Sim direction specifically.

**The reverse direction is also automatic, now confirmed**: `_training_loop()` calls `_save_agent()` — which writes straight back to the same `agent_dir / "brain.pcap"` path the agent was loaded from — every 10 episodes (`if (episode + 1) % 10 == 0`), plus once more unconditionally right after training completes. Nothing manual about either direction: drop a capsule in, it trains, the same file periodically gets overwritten with the updated brain, full round-trip.

Comment in the source notes the watcher deliberately replaced an earlier `carb.events.Type.FILESYSTEM`-based approach that was broken — simple directory polling instead, on a plain background thread.

## A live upstream bug, documented rather than silently worked around

The header calls out a known Isaac Sim issue directly (GitHub #320): `ArticulationController.apply_action()` raises a `TypeError` when the world backend is `"torch"` and `joint_velocities` is `None`. The stated fix/workaround: pass only `joint_positions` _or_ `joint_velocities` in a given call, never a mixed array containing `None` values. Worth knowing if `IsaacSimActuator`'s joint-command code is ever touched — this is an external constraint, not something fixable on the DivineWorld side.

## Design discipline worth noting

The header states explicitly: "No subclassing of internal base classes — uses real public APIs only." Given Isaac Sim's own API churn (documented in the same file, 4.x → 5.0), this is a sensible defensive choice — subclassing internal/private classes tends to be exactly what breaks hardest across a major version bump.