---
type: file
status: ingested
---

# cognitive_loop.py

💡 **Role**: the actual autonomous heartbeat — runs Perceive→Think→Reflect→Decide→Act every `loop_interval` seconds. See [[cognitive-loop]] for the five-step cycle's narrative, the deliberation-gating philosophy, plan-interruption logic, the specialization/learning-mode signal system, and self-referential skill practice — all in real depth there already. This page: the mechanical catalog (every method, one line each), `CognitiveState` (not covered on the topic page at all), the broadcast/input-reception plumbing, and `PLAN_BLEND` — the one genuinely new mechanic from commit `8839776`, not yet on the topic page since it postdates it.

## Imports

`asyncio`, `time`, `numpy`, `typing`, `collections.deque`, `logging` — all external/stdlib. No internal project imports at module level. Lazy, inside specific methods: `from ai_core.brain_core import _action_to_vector` (inside `_record_skill_attempt()` and `_execute_planned_action()` — see [[brain_core]]), `from ai_core.planner import ACTION_TYPE_INDEX` (fallback inside `_record_skill_attempt()` only, if the `brain_core` import fails), `import hashlib` (stdlib, inside `_think()`), `from chat_system import chat_system` (inside `_execute_file_processing()`, wrapped in its own silent `except`).

## Module-level functions

- `_event_to_text(event)` — serializes a non-chat event dict (audio, action, perception, generic) into a short natural-language sentence, e.g. `{'type': 'chat_heard', 'payload': {'speaker': 'Eve', 'message': 'hello'}}` → `"Eve said: hello"`. Returns an empty string for event shapes it doesn't recognize rather than guessing. See [[cognitive-loop]] for why this exists (training the language system on the full experience stream, not just chat).
- `_get_dominant_event(agent)` — the most frequent `type` string across the last 30 memory events, used as a self-attributed "domain" label. Independently confirmed against `agent.py` that `agent.memory.events` is a plain list attribute, not a method — this function would silently return `None` on every call if that assumption were wrong, so it's worth knowing it actually checks out.

## Classes

### `CognitiveState`
Not mentioned on [[cognitive-loop]] at all — plain per-agent state, all primitive fields, no methods beyond `__init__`: current focus/goal, attention/energy levels (energy decays 0.05% per cycle, floored at 0.3 — read by `_reflect()`'s action-cooldown scaling, described below), the last significant event, cycle/speech counters, and the three fields that make mid-plan interruption possible (`current_plan`, `current_plan_score`, `plan_step`).

### `CognitiveLoop`
**Class constant**: `PLAN_BLEND = 0.25` — see "Plan-blending" below; not yet on [[cognitive-loop]] or [[imagination-and-planning]] since it was added this commit.

**Control:**
- `__init__(agent, loop_interval)` — five bounded deques (visual/audio/state/event/file buffers), a `CognitiveState`, timing trackers, the 5-cycle curiosity+surprise streak counter feeding learning-mode entry (see [[cognitive-loop]]), and the per-domain reward history dict feeding the specialization signal (also [[cognitive-loop]]).
- `_update_thresholds()` — derives speech/action/learning cooldowns from personality traits (extraversion+sociability lower the speech cooldown; boldness lowers the action cooldown; openness lowers the learning interval).
- `start()` *(async)* — spawns `_cognitive_loop()` as a background task; no-ops with a warning if already running.
- `stop()` *(async)* — cancels the task and awaits its `CancelledError`.

**Main loop:**
- `_cognitive_loop()` *(async)* — the actual `while self.running` cycle: perceive → think → reflect → decide → act → update state, sleeping out whatever's left of `loop_interval` after accounting for how long the cycle took. A bare exception anywhere in one cycle is logged and swallowed with a 1-second backoff sleep, rather than killing the loop.

**Perception:**
- `_perceive()` — builds the base perception dict (health/hunger/emotions/memory size, recent events, novelty from event-type frequency, urgency from emotion intensity vs. health deficit), then augments it with a `VisionAdapter`'s latest visual token and blends in visual novelty if one is attached.

**Thinking** *(the biggest method in the file — `_think()`, async, ~330 lines)*:
- Runs `brain.evaluate_event()` unconditionally, then quick-exits on a routine cycle (low novelty, low urgency, no queued file) before anything expensive.
- Audio: if an `audio_processor` is attached, transcribes one chunk off-thread, routes any transcription through `brain.process_language_input()` tagged `speaker_id='nearby_voice'` (a real fixed bug: this call was missing `speaker_id` entirely, which would now raise, since the underlying method requires it for familiarity tracking), and broadcasts both the heard audio and any reply.
- Calls `brain.should_deliberate()` / `brain.deliberate()` — see [[cognitive-loop]] for the two-speed philosophy; this is the only call site for `deliberate()` in the whole codebase.
- Autonomous listening and "verbal thinking" (generating and broadcasting an internal thought) are each gated by their own personality-weighted desire score, checked stochastically (`random() < desire`), not a hard threshold.
- Computes `language_desire` from a weighted mix of emotion intensity, novelty, personality, time-since-last-speech, and active conversation context.
- File-processing check: promotes a `file_uploaded` memory event to the front of `file_buffer` if no matching `file_processed` event already exists — the actual retry/deduplication logic behind file uploads.
- Curiosity score, web-browsing gate (via `brain.should_browse()`), and focus determination (single `if/elif` chain: survival > learning > planning > expression > exploration > observation).
- Learning-mode streak logic: on entering or exiting `PolicyBridge` learning mode, calls `continual_learner.switch_task()` with an MD5-derived task ID (see [[cognitive-loop]] for why not Python's built-in `hash()`) — this file is where that call actually happens, not `continual_learner.py` itself.

**Reflection:**
- `_reflect(thoughts, perception)` — turns the raw `thoughts` dict into boolean should-I flags, in a fixed priority (file > speech > action > browse > learn implied by return order, though `_decide()` is what actually enforces priority — see below). Notably: `effective_cooldown = action_cooldown / max(0.3, energy_level)` — a genuinely fixed bug, since `CognitiveState.energy_level` decayed every cycle but nothing previously read it, so a "tired" agent acted exactly as fast as a fresh one.

**Decision:**
- `_decide(reflection, thoughts)` — the actual priority enforcement via `if/elif` order: `process_file` (1.0) > `speak` (0.9) > `action` (0.7) > `learn` (0.3) or `web_browse` (0.6, checked last despite the higher number, since it's in the final `elif`). Also where `agent._last_deliberation`/`_last_deliberation_obs` get persisted for the continual-learning worker to pick up later in the same cycle.

**Action execution:**
- `_act(decision, perception, thoughts)` *(async)* — dispatches on `decision['type']`; the `action` and `learn` branches run their executor concurrently with `_execute_learning_async()`/`_execute_continual_learning_async()` via `asyncio.gather(..., return_exceptions=True)`, so one failing task can't block the others.
- `_execute_action(perception, deliberation)` *(async)* — builds a plan (deliberation's best action spliced into `planner.generate_plan()`'s output if available, else a fresh `generate_plan()` call, else a single unplanned action via `_action_worker`), then steps through it one action at a time via `asyncio.to_thread`, checking for hard interruption (urgency ≥ 0.75) and soft interruption (a fresh deliberation beats the running plan) between every step — see [[cognitive-loop]] for the interruption philosophy itself.
- `_record_skill_attempt(step)` — see [[cognitive-loop]]'s self-referential skill practice section for the concept; mechanically, this is the method that actually calls `skill_tracker.record_attempt()`, using `_action_to_vector()` (falling back to a raw one-hot if the import fails) to encode the step actually taken, compared against `_last_deliberation.best_action_vector`. On low-progress frustration crossing `0.7 × persistence`, this is also where the focus task gets abandoned and `continual_learner.switch_task(0)` fires.
- `_execute_planned_action(action, context)` — **`PLAN_BLEND` lives here.** Computes `action_arr` from the raw policy via `agent.decide()`, then linearly interpolates it `(1 - PLAN_BLEND) * action_arr + PLAN_BLEND * planned_vec` (clipped to `[-1, 1]`) before sending anything to Minecraft — `planned_vec` being `action` (this plan step) run through `_action_to_vector()`. Before this fix, `action` was computed and then used only for a log line; the plan chosen by `deliberate()`/`generate_plan()` never touched what was actually sent to the game. Also where the god-vs-NPC `act_god()`/`act()` routing lives (gods need the 18-dim call to get ability dims 13–17 decoded; a previous version always used the 13-dim `act()`, silently dropping ability output for every god).
- `_action_worker(context)` — the same decide→route→send logic as above, minus any plan or blending, for the no-plan fallback case.

**Speech:**
- `_execute_autonomous_speech(perception, thoughts, content)` *(async)* — guards against `brain.language` being `None` (a fixed bug: this was the one speech call site missing that check), generates speech via `brain.language.generate_speech()`, remembers it, and broadcasts it.

**File processing:**
- `_execute_file_processing(file_info)` *(async)* — runs `brain.learn_from_file()` off-thread, remembers the result, and best-effort notifies `chat_system` (any failure there is silently swallowed). Retries up to 3 times (re-appending to `file_buffer`) on exception.

**Web browsing:**
- `_execute_web_browsing()` *(async)* — visits up to 2 queued pages per cognitive cycle via `browser.browse()`. A fixed bug worth knowing: this used to peek (`browse_queue[0]`) instead of dequeue, so the same URL would be re-browsed forever. Also computes `claim_support_delta` (a before/after memory-search count around the current conversation topic) and fires it as a `web_browsed` reward event — the mechanism behind [[reward-and-learning-stack]]'s evidence-checking reward term.

**Learning:**
- `_execute_learning_async()` *(async)* — thin wrapper running `_learning_worker()` off-thread.
- `_learning_worker()` — see [[cognitive-loop]] for why non-chat events get serialized via `_event_to_text()` before training; mechanically, every such synthetic sample is tagged `speaker_id='_internal_experience_replay'`, a deliberately distinct bucket from real chat so it can never inflate genuine repeat-visitor familiarity scoring.
- `_execute_continual_learning_async()` *(async)* — thin wrapper running `_continual_learning_worker()` off-thread.
- `_continual_learning_worker()` — snapshots `continual_learner.experience_buffer` *before* calling `learn_from_buffer()` (a fixed bug: `learn_from_buffer()` clears the buffer itself, so reading it afterward — as an earlier version did — always saw an empty list). The snapshot feeds `SelfSupervisedTrainer.train_step()`; the most recent deliberation's `all_scored_actions` feeds `policy.grpo_update()` (defined in [[reward-and-learning-stack|self_supervised_trainer.py]]) if both are available. `grpo_update()`'s returned `mean_advantage` is persisted onto the agent by the caller here, then folded into `_domain_reward_history` for whatever domain `_get_dominant_event()` currently identifies.

**Focus-task resolution** (mechanism behind [[cognitive-loop]]'s two-path specialization system):
- `_resolve_active_focus_task()` — Path 2 (`_check_specialisation_signal()`) checked first, Path 1 (`skill_tracker.get_weakest_task()`) as fallback.
- `_check_specialisation_signal()` — a domain qualifies once it has ≥50 reward samples and its most recent 50-sample mean beats the pre-existing baseline by more than one standard deviation; a domain that's plateaued (variance < 0.02 over its last 30 samples) is excluded even if it once qualified.

**State update:**
- `_update_cognitive_state(perception, thoughts, reflection)` — updates focus/attention/energy, records a "significant event" snapshot above novelty/urgency 0.7, and decays emotions.

**Input reception** (none of these mentioned on [[cognitive-loop]]):
- `receive_visual_input(frame, metadata)` / `receive_audio_input(audio, sample_rate, metadata)` / `receive_state_update(state)` / `receive_file(file_info)` — the external entry points other code calls to push sensor data and health/hunger updates into this loop; the first two just remember a lightweight metadata record (shape/duration, not the raw frame) rather than processing the data directly here.

**Broadcasting** (all four also not detailed on [[cognitive-loop]]; see [[electron-frontend]] for how the frontend consumes each message type):
- `_broadcast_speech(speech)` *(async)* — sends a `chat`/`from:agent` message over the agent's broadcast channel and, separately, pushes the same text into in-world Minecraft chat if `_minecraft_send_chat` is available.
- `_broadcast_internal_thought(thought)` *(async)* — sends `agent_thought`, prefixed with 💭.
- `_broadcast_mental_workspace(workspace_data)` *(async)* — sends `visualization_update` — this is what [[electron-frontend]]'s Mental Workspace panel object-count summary is built from.
- `_broadcast_audio_heard(transcription, emotion)` *(async)* — sends a `chat`/`from:system` message with an emotion-keyed emoji prefix.

**Status:**
- `get_status()` — a flat dict snapshot (running state, cycle/speech counts, current focus, attention/energy, cooldowns, queued-file count, active-plan info) for external status reporting.

## Problems (faced by traditional AI systems / LLMs)

Two distinct, well-known problems this file addresses at once. First: an agent loop that always runs its most expensive reasoning path on every tick either can't keep up in real time or wastes most of its compute budget on uninteresting moments — the standard motivation for any two-speed or hierarchical control architecture. Second, and more specific to plan-following agents: a planner that computes a sequence but has no mechanism to actually influence the low-level controller's output is a planner in name only — a well-known gap between "what the agent decided" and "what the agent's body actually does," here it was previously bridged by nothing more than a debug log line.

## Solutions

The five-step cycle with `should_deliberate()`'s gate is the answer to the first problem (detailed on [[cognitive-loop]]). `PLAN_BLEND` is the answer to the second: rather than letting the plan fully override the policy's output — which the file's own `planner.py` docstring explicitly argues against ("the agent chooses freely") — a bounded linear interpolation gives the plan real, proportional influence over what gets sent to Minecraft without removing the policy's freedom to deviate. Both fixes follow the same "bias, not restriction" principle this commit applies throughout: nudge the outcome, never hard-cap the option space.

## Files Required

- `brain_core.py` — `_action_to_vector()`. Not yet given its own file-level page — see [[brain-core]] for the narrative treatment in the meantime. *(Also see [[brain_core]] for this same file's own mechanical catalog.)*
- `planner.py` — `ACTION_TYPE_INDEX` (fallback only) and `CognitivePlanner.generate_plan()`. Not yet given its own file-level page — see [[imagination-and-planning]]. *(Also see [[planner]] for its mechanical catalog.)*
- `hashlib` and `chat_system` — external/sibling modules, not part of `ai_core/`.

## Files Used In

- `agent.py` — constructs and owns a `CognitiveLoop` instance; see [[agent-runtime]] for the narrative treatment.
- `skill_tracker.py` / `policy_bridge.py` / `self_supervised_trainer.py` — all read from or written to by the continual-learning and skill-practice methods above. None yet has its own file-level page — see [[reward-and-learning-stack]] in the meantime.
