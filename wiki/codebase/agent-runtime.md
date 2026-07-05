---
type: system
status: ingested
---

# Agent Runtime (`NPCAgent`)

💡 **What this is**: `ai_core/agent.py`'s `NPCAgent` class — the object every Civilian Agent and God Agent actually is, at the code level. One instance per agent, one FastAPI process per instance (see [[architecture-overview]]).

## Construction order matters

`__init__` builds subsystems in a specific sequence, and several fixes in this codebase exist specifically because that order was wrong at some point:

1. **Personality + Emotion** — see [[personality-and-emotion]]. Gender assigned via `assign_npc_gender()` (random male/female) unless it's a god, which gets `assign_god_gender()` → always `'dual'`.
2. **Memory** — `UnifiedMemoryStore` (capacity 10,000, optional ScyllaDB backend). See [[memory-and-braincapsule]].
3. **BrainCore + CognitivePlanner** — the deliberation/evaluation engine and the plan generator. `BrainCore` isn't deep-ingested into its own page yet, but its public surface (`evaluate_event`, `should_deliberate`, `deliberate`, `set_world_model`, `set_reward_system`) is used constantly across [[cognitive-loop]] and [[reward-and-learning-stack]].
4. **Language** (`brain_language.add_language_to_brain`) — attached to the brain, not the agent directly.
5. **Continual learning** (`add_continual_learning(self, strategy='replay')`) — Avalanche-based, see [[reward-and-learning-stack]].
6. **`obs_dim`/`action_dim`** set explicitly (128 / 13-or-18) — a fixed bug: these were previously never set on the agent at all, so every downstream `hasattr` guard checking for them silently failed and skipped attaching PolicyBridge/SelfSupervisedTrainer/SkillTracker for *every single agent*.
7. **RewardSystem** — initialized eagerly, not lazily, specifically so `brain.reward_system` is never `None` mid-session.
8. **World model** — `WorldModel` or `EnsembleWorldModel` (5 members), see [[reward-and-learning-stack]].
9. **Vision, then audio processing.**
10. **Cognitive loop** — only if `autonomous_mode` is true. A chat-mode agent never constructs one at all (see below).
11. **Policy auto-init** — `TransformerPolicy` (13-dim) or `GodTransformerPolicy` (18-dim), so `decide()` never silently falls back to a random action vector.
12. **PolicyBridge / SelfSupervisedTrainer / SkillTracker attachment** — gated on policy, continual_learner, obs_dim, and action_dim all being present. This used to live only in a dead code path (`agent_runner.py`, never actually invoked by the packaged launcher) — moved into `__init__` directly so every agent gets it regardless of entry point.
13. **Web browsing, Minecraft client (if `mode='minecraft'`), god controls (if `god_type` is set).**

## Three modes

- **`autonomous`** — full `CognitiveLoop` running, no Minecraft connection required. This is what lets an agent keep living/learning with no robot body or game client attached, matching the design wiki's Layer 1 philosophy.
- **`minecraft`** — `CognitiveLoop` *and* a `MinecraftClient` (TCP+WS dual transport, see [[communication-protocol]]).
- **`chat`** — no `CognitiveLoop` at all. Instead, `start_autonomous_speech()` runs a lightweight timer-driven loop that periodically asks the language system `should_speak()` and broadcasts if so. Built specifically for the packaged chat-mode launcher and any text-only deployment — has no Minecraft perception/action loop in it whatsoever.

## Action space

Two sizes, both built from the same base:

- **13-dim base** (`TransformerPolicy.BASE_DIM`): move_forward, move_strafe, jump, sneak, attack, use, drop, open_inv, swap_hand, yaw_delta, pitch_delta, sprint, hotbar_slot.
- **18-dim god** (`GodTransformerPolicy.TOTAL_DIM`): the same 13, plus 5 more — trigger_flag, ability_idx, param1, param2, param3 — decoded by `act_god()`, not `act()`. A god agent whose action array is routed through the wrong method silently loses its ability dims; this was a real bug (abilities never fired because the wrong dims were being read as trigger/ability_idx) that's now fixed by checking `god_type` explicitly at the call site.

## Save / load — what actually gets persisted

`save(path)` builds a `BrainCapsule` (see [[memory-and-braincapsule]] for the full field list) and separately serializes a `model_state` dict containing: policy weights, world model (config + weights), vision vocab, RewardSystem's RND/ICM network weights, up to the last 10,000 `EpisodicMemory` transitions, and `WorldModelTrainer`'s step count + loss history. `load(path)` reverses all of this, including re-integrating god controls if `god_type` was persisted (a fixed bug: it previously wasn't, so a restarted god agent lost its ability space and fell back to a plain 13-dim policy).

**Save cadence**: `run_standalone_agent()`'s main loop saves every **300 seconds** (5 minutes) during a live session, plus once immediately at startup (before the server even starts accepting connections — fixed specifically to avoid a race where a client could request `/status` before `brain.pcap` existed) and once on `shutdown()`.

> **Cross-wiki note**: the design wiki's [[wiki/design/Robots|Robots]] page (Compute Architecture section) states a "roughly one-minute" save cadence as a *recommendation* for the physical-robot build. The actual current code saves every 5 minutes, not 1. This isn't a bug — the design wiki is describing a target for hardware that doesn't exist yet, not this code — but it's worth knowing the two numbers don't match today, and worth tightening the interval here if the 1-minute target is adopted for real.

## HTTP/WS surface

Each agent's FastAPI app exposes, roughly: `/status`, `/thoughts`, `/chat` (with a `speaker_id` field specifically added to support genuine repeat-visitor familiarity in the reward system — see [[reward-and-learning-stack]]), a full `/browser/*` set (navigate/click/type/scroll/screenshot/history/allowed_sites) for the agent's own attached browser, `/api/controller/*` for the human-operator "controller" mode (camera/mic passthrough for multimodal learning), and `/api/upload` for dropping files an agent should learn from. `/ws` is general chat; `/ws/agent` is the binary perception/action channel the Java mod actually uses.
