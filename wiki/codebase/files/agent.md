---
type: file
status: ingested
---

# agent.py

💡 **Role**: two things in one file — the `NPCAgent` class every Civilian/God Agent actually is, and a standalone runnable module (its own FastAPI app plus a `run_standalone_agent()`/CLI bootstrap) that can run one agent on its own, without `main.py`. [[agent-runtime]] already covers `NPCAgent`'s 13-phase construction order, the three modes, the 13/18-dim action space, and save/load in real depth — kept terse here. This page's actual additions: the module-level routes as concrete handlers (not just a list of paths), the standalone-runner functions (not on the topic page at all), and two fixes the topic page doesn't mention.

## Imports

External: `fastapi`, `uvicorn`, `argparse`, `signal`, `json`, `time`, `asyncio`, `logging`, `pathlib.Path`, `typing`, `numpy`, `torch`, `sys`. Internal, at module level: `from ai_core.personality import ...`, `from ai_core.emotion import ...`, `from ai_core.memory import UnifiedMemoryStore` (see [[memory-and-braincapsule]]), `from ai_core.brain_core import BrainCore`, `from ai_core.planner import CognitivePlanner`, `from ai_core import brain_language`, `from ai_core.continual_learner import add_continual_learning`, `from ai_core.reward_system import RewardSystem`, `from ai_core.world_model import WorldModel, EnsembleWorldModel`, `from ai_core.vision import VisionAdapter`, `from ai_core.audio_processors import AudioProcessor`, `from ai_core.cognitive_loop import CognitiveLoop`, `from ai_core.policy_bridge import PolicyBridge`, `from ai_core.self_supervised_trainer import SelfSupervisedTrainer`, `from ai_core.skill_tracker import SkillTracker`, `from ai_core.memory import BrainCapsule` (or equivalent capsule type — see [[memory-and-braincapsule]]). Lazy, inside specific methods: `from rl.policy import TransformerPolicy` / `GodTransformerPolicy` (inside `initialize_policy()`), `from ai_core.god_controls import GodControlSystem` (same method, god branch only), `from py_backend.utils.dw_controller import ControllerRuntime` (inside `_get_controller()` — a file not yet read or given any page, this pass's one open thread).

## Module-level FastAPI app & routes

A second, separate app instance from `main.py`'s — this is the **per-agent** server, bound to the agent's own `backend_port`, because the agent's own browser/controller instances live on `self`, not on any shared manager. [[electron-frontend]] and [[human-controller-debug]] both talk to *this* app, not `main.py` — worth knowing since neither of those pages currently says so explicitly.

- `GET /status`, `GET /thoughts` — plain status/thought-log reads off `global_agent`.
- `POST /chat` — the `speaker_id` field (see [[electron-frontend]]'s `VISITOR_ID`) is what feeds [[reward-and-learning-stack]]'s familiarity reward term.
- `POST /api/agents/{agent_id}/web/allow` — updates the attached browser's allowed-domain set.
- `POST/GET /browser/navigate|click|type|scroll|screenshot|stats|history|allowed_sites` — direct pass-through to `agent.browser`, a `WebBrowser` instance (not yet given its own page).
- `GET /api/controller/detect-devices`, `POST /api/controller/activate`/`deactivate`, `GET /api/controller/status` — this is the *real* "Controller Mode" [[electron-frontend]]'s `ControllerSafety.jsx` UI is for (camera/mic/filesystem/network access for the AI itself) — confirmed here as backed by `ControllerRuntime` from `py_backend/utils/dw_controller.py`, lazily imported per-request inside `_get_controller()`. Entirely separate code from [[human-controller-debug]]'s `human_controller_server.py`, despite both using the literal string "controller" — two unrelated systems, now confirmed from both directions.
- `POST /api/upload` — writes an uploaded file to disk and calls `agent.brain.learn_from_file()`, optionally synchronously (`sync=true` in the form) or as a background task.
- `WS /ws` — general chat/thought/activity broadcast (see [[electron-frontend]]).
- `WS /ws/agent` — the binary perception/action channel the Java mod actually uses (see [[communication-protocol]]).

## Classes

### `NPCAgent`
See [[agent-runtime]] for construction order, modes, action space, and save/load — not repeated here. Additional methods not covered there:

- `perceive()` / `observe_environment(observation)` — pull the latest state into the agent and hand it to `cognitive_loop`/`brain` as appropriate.
- `process_chat(message, speaker_id)` — the non-HTTP entry point `/chat`'s handler and the Minecraft chat path both eventually call; routes to `brain.process_language_input()`.
- `imagine_scenario(sequence, context)` / `generate_internal_thought()` — thin pass-throughs to `planner.imagine_sequence()` and `brain.language`'s thought-generation, respectively.
- `use_god_ability(name, outcome, **params)` — called from `act_god()` below (and from admin commands — see [[commands]]); delegates to `god_controls`.
- `initialize_policy(obs_space, action_space)` — builds `TransformerPolicy` or `GodTransformerPolicy` depending on `god_type`, sized 13 or 18 dims respectively — see [[agent-runtime]] for why the sizes matter.
- `decide(obs, deterministic)` — **not on the topic page**: a real, load-bearing fix lives here. Previously the actual action vector always came from `policy._predict()` directly, completely bypassing `PolicyBridge` regardless of whether [[cognitive_loop]]'s N=5 curiosity streak had switched learning mode on — meaning `cl_head` was built, soft-synced, and toggled, but never actually ran a real decision. Now checks `policy_bridge.is_in_learning_mode()` first and routes through `bridge.predict_action()` when true, falling back to the direct policy call otherwise (or a randomized 13-dim vector if no policy exists yet at all).
- `act(action)` / `act_god(action)` — see [[agent-runtime]]'s action-space section for the dimension layout and its bug history; not repeated here.
- `shutdown()` — stops `cognitive_loop` (if running) and any autonomous-speech loop, then saves.
- `save(path)` / `load(path)` — see [[agent-runtime]] and [[memory-and-braincapsule]].

No async methods on `NPCAgent` itself — `start_autonomous_mode()`/`start_autonomous_speech()`/`stop_autonomous_speech()` (mentioned on [[agent-runtime]]) are the only ones, and their behavior is already covered there.

## Functions & Async Functions (module-level, outside the class)

- `run_server(port)` *(async)* — the actual `uvicorn.Server` runner for this file's own FastAPI app.
- `run_standalone_agent(...)` *(async)* — constructs an `NPCAgent` directly (bypassing `main.py` entirely) and drives it through one of three modes:
  - `autonomous` — starts `CognitiveLoop` via `start_autonomous_mode()`.
  - `minecraft` — **a real fix, not on the topic page**: this branch used to start only the TCP-fallback action loop and wait for a Minecraft connection, never actually starting `CognitiveLoop` — meaning deliberation/GRPO/skill-tracking silently never ran for any agent launched this way. Now calls `start_autonomous_mode()` first (non-blocking), *then* starts the TCP loop as the documented WS-disconnection fallback it was always meant to be, then checks for a bundled UltimMC launcher and waits (up to 300s) for the mod to connect.
  - `chat` — **also a real fix**: previously did nothing at all unless the separate `chat_interface` console flag was also set. Now unconditionally starts `start_autonomous_speech()` (a lightweight timer-driven engagement loop, not `CognitiveLoop`, which has no meaning without Minecraft perception) as a background task regardless of the console-chat flag.
  - All three modes then enter a periodic-save loop (every 300s, timestamp-based rather than a modulo check — a fixed bug, since `% 300` on a drifting `asyncio.sleep(1)` can skip the exact boundary under event-loop load) until `duration` elapses or `KeyboardInterrupt`.
- `chat_loop(agent)` *(async)* — the interactive stdin console reader, gated behind `--chat`, distinct from `chat` mode itself.
- `main()` — the CLI entry point (`argparse`, ~15 flags mirroring `run_standalone_agent()`'s parameters), calling `asyncio.run(run_standalone_agent(...))`.

## Problems (faced by traditional AI systems / LLMs)

Any system with more than one code path capable of producing "the" action (here: a direct policy call and a `PolicyBridge`-mediated call) risks one of them being wired up but never actually reached — every symptom of that path being built looks correct (weights update, flags toggle) while the behavior it was meant to produce never happens. Separately: a multi-mode entry point (autonomous/minecraft/chat here) tends to accumulate exactly one mode that was fully implemented and tested, while the others silently do less than their name promises — a general risk of any branch that's easy to add but easy to leave half-finished.

## Solutions

Both problems get the same kind of fix here: trace the actual call graph for each mode/path rather than trusting what the surrounding comments and code structure imply happens, and make the correct behavior the only behavior (route `decide()` through `PolicyBridge` when the condition is met, full stop; call `start_autonomous_mode()` unconditionally at the top of every mode branch that needs it) rather than leaving a second, easier-to-reach path that quietly wins by default.

## Files Required

- [[memory-and-braincapsule]]-relevant: `memory.py` (`UnifiedMemoryStore`, capsule type).
- `brain_core.py`, `planner.py` — see [[brain_core]] and [[planner]] for their own mechanical catalogs.
- `brain_language.py`, `continual_learner.py` (see [[continual_learner]]), `reward_system.py`, `world_model.py`, `vision.py`, `audio_processors.py`, `cognitive_loop.py` (see [[cognitive_loop]]), `policy_bridge.py`, `self_supervised_trainer.py`, `skill_tracker.py` — none except `continual_learner.py`/`cognitive_loop.py` yet has its own file-level page; see [[reward-and-learning-stack]]/[[perception-and-actuation]]/[[language-system]] for narrative coverage in the meantime.
- `rl/policy.py` — `TransformerPolicy`/`GodTransformerPolicy` (lazy import).
- `ai_core/god_controls.py` — `GodControlSystem` (lazy import, god branch only).
- `py_backend/utils/dw_controller.py` — `ControllerRuntime` (lazy import inside `_get_controller()`) — **not yet read this pass**; the one open thread from this file's own ingest.

## Files Used In

- `main.py` — spawns this file's standalone-agent logic as a subprocess per agent, rather than importing it directly (confirmed by the module's own CLI/`argparse` design, built to be launched, not imported).
- Every Java-mod-side TCP/WS client (see [[communication-protocol]]) talks to whichever port this file's `NPCAgent` binds.
- [[electron-frontend]] and [[human-controller-debug]] both call this file's own FastAPI routes directly, as noted above.
