---
type: file
status: ingested
---


# chat_system.py

💡 **Role**: a routing-only layer between GUI/game WebSockets and `agent.brain` — its own module docstring is explicit that it does no language processing or decision-making itself. **Open question, not resolved this pass**: every call into `agent.brain` here is guarded by `hasattr()` with a `_fallback_*` method behind it, which reads like defensive code written for an `NPCAgent`/`BrainCore` interface that might not always have `process_language_input`/`learn_from_file`/`should_speak`/`generate_speech` — but everything read elsewhere in this ingest confirms `BrainCore/LanguageIntelligence` _does_ always expose those methods once attached. Whether `main.py` (not yet read) actually wires this module in as the live routing path, or whether it's an earlier/parallel layer largely superseded by `agent.py`'s own direct FastAPI routes and `communication_protocol.py`'s WebSocket handlers, wasn't confirmed.

## Imports

`asyncio`, `json`, `logging`, `time`, `collections.deque`, `dataclasses`, `pathlib.Path`, `typing`, `numpy`, `websockets.exceptions.ConnectionClosed` — all external/stdlib. No internal project imports — this file only ever touches `agent.brain`/`agent.memory`/`agent.emotion` via duck typing, never importing their classes.

## Classes

### `ChatMessage` (dataclass)

`text`, `timestamp`, `expires`, `sender`, `is_emote`, `bubble_height` — the exact shape sent to both GUI and game WebSockets.

### `UnifiedChatSystem`

**Methods:**

- `__init__()` — five parallel dicts/sets keyed by `agent_id`: message queues, GUI/game connections, which GUIs are active, entity type (god vs. npc, for bubble height), and running autonomous-speech tasks.
- `register_agent(agent_id, agent)` / `register_gui` / `unregister_gui` / `register_game` / `unregister_game` — connection bookkeeping; `unregister_gui` also cancels that agent's autonomous-speech task, so closing the GUI is what actually stops the speech loop.
- `send_message(agent_id, message, target, sender, is_emote, bubble_height, expire_after)` — the outbound path. Bubble height defaults 3.0 for gods / 2.0 for NPCs if not given explicitly; `is_emote` auto-detects `*wrapped in asterisks*` text even if not explicitly flagged. Every sent message is also queued (capped at 100 per agent) regardless of whether either target connection currently exists.
- `_send_to_game(agent_id, msg)` / `_send_to_gui(agent_id, msg)` — the two actual WebSocket sends, each independently try/excepted so one target failing never affects the other.
- `handle_user_message(agent_id, message)` — inbound GUI chat. Prefers `agent.brain.process_language_input(message, context)`, falls back to `_fallback_response()` if that method isn't present.
- `handle_file_upload(agent_id, file_data, filename, filetype)` — writes to `data/uploads/{agent_id}/{filename}` (a different path convention from `agent.py`'s own `/api/upload` route — not reconciled this pass whether both paths are actually live simultaneously), then prefers `agent.brain.learn_from_file()`, falling back to `_fallback_file_learning()`.
- `handle_minecraft_perception(agent_id, frame, audio, action, game_state)` — buffers frame/audio into `agent.perception_buffer` with timestamps, prefers `agent.brain.process_perception()` (a method name not encountered anywhere else in this entire ingest pass — not confirmed to exist on `BrainCore`), falling back to `_fallback_perception()`. Also checks `should_speak()`/`generate_speech()` afterward, independent of whichever perception path ran.
- `start_autonomous_speech(agent_id)` / `stop_autonomous_speech(agent_id)` — a simple 5-second-interval loop (10s backoff on error) checking `brain.should_speak()` while either a GUI or game connection is active; a second, independent implementation of "autonomous speech" from [[cognitive_loop]]'s own `_execute_autonomous_speech()` — not confirmed whether both run simultaneously in practice or this is a legacy/alternate path.
- `send_thought(agent_id, thought, is_internal)` / `send_visualization(agent_id, data)` — direct GUI-only pushes, bypassing the queue/game-routing logic `send_message()` uses.
- `is_gui_active(agent_id)` — plain membership check.
- `_build_context(agent)` — health/hunger/emotion snapshot/dominant emotion/memory size, plus perception buffer and last action if present — a different, smaller context shape than [[brain-core]]'s own `_perception_to_context()`.
- `_fallback_response()` / `_fallback_file_learning()` / `_fallback_perception()` — plain, simple implementations (remember the event, return a canned string) used only when the preferred `agent.brain` method is absent.

No other classes.

## Functions

- `handle_frontend_chat(agent_id, message)` / `handle_frontend_file_upload(...)` / `handle_minecraft_frame(...)` / `start_agent_autonomous_speech(agent_id)` — thin `async` wrappers around the module-level `chat_system` singleton's own methods, per the module docstring's stated "Public API (used by main.py / FastAPI endpoints)."

## Module-level singleton

- `chat_system = UnifiedChatSystem()` — one instance, constructed at import time, shared by every wrapper function above.

## Problems (faced by traditional AI systems / LLMs)

A routing layer written defensively against an interface that might not always be fully implemented (via `hasattr()` checks and fallbacks) is a reasonable design when the calling code genuinely can't assume what it's talking to — but if the actual, current `agent.brain` always implements every method being checked for, the defensiveness adds a second, simpler code path that's easy to lose track of, and — as the `should_speak()`/`_execute_autonomous_speech()` overlap here shows — risks two independent implementations of "the same" behavior running (or half-running) at once.

## Solutions

Not resolved in this file — this is exactly the kind of open architectural question worth a direct answer from Devlord rather than a guess: is this module still the live routing path (in which case the fallbacks are dead code worth removing, and the autonomous-speech duplication with `cognitive_loop.py` is worth reconciling), or is it superseded by `agent.py`'s direct routes and `communication_protocol.py`'s WebSocket handlers (in which case the whole file might be one to flag on [[known-issues]] rather than treat as current)?

## Files Required

None — every interaction with `agent.brain`/`agent.memory`/`agent.emotion` is duck-typed, never imported.

## Files Used In

- Presumably `main.py`, per this file's own docstring — not yet confirmed, since that file isn't read this pass.