---
type: file
status: ingested
---
# chat_system.py

💡 **Role**: a routing-only layer between GUI/game WebSockets and `agent.brain` — its own module docstring is explicit that it does no language processing or decision-making itself. **Resolved — see [[main]] for the full trace**: this module is not the live routing path. `main.py` never imports it (zero references, confirmed by direct search); `agent.py` never imports it either, and its own `/ws`/`/chat` routes call `process_chat()` directly with no `hasattr()` guarding of any kind; `auto_connect_system.py` doesn't touch it. The one surviving reference anywhere in the codebase is a single `try/except`-guarded call from [[cognitive_loop]]'s `_execute_file_processing()` — and even that call is functionally inert: this file is explicitly excluded from the packaged-agent build (`AGENT_EXCLUDE_MODULES`, both `config.py` copies), each agent subprocess would get its own disconnected `UnifiedChatSystem` singleton even when the import succeeds (subprocess-per-agent means no shared memory), and nothing anywhere ever calls `register_gui`/`register_game` to give it a live connection to deliver to in the first place. The `hasattr()` defensiveness that originally read as unexplained paranoia may have a real explanation, not yet confirmed: `main.py`'s `/api/agents/chat_heard` docstring states god agents run a separate `LLMOracleBrain`, not `BrainCore`/`LanguageIntelligence` — if accurate, this file's guards may have been written for that split rather than for no reason at all.

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

Resolved this pass (full trace on [[main]]): this module is dead code, not a functioning parallel system. The `hasattr()` fallbacks are inert — nothing calls `handle_user_message`/`handle_file_upload`/`handle_minecraft_perception` from outside this file at all, so the code paths they guard never run either way. The autonomous-speech duplication with [[cognitive_loop]] flagged as a risk in Problems above is not actually a live duplication in practice: this file's own loop is never started, since `start_autonomous_speech()`/`register_gui()` have no external call sites. Whether to formally retire this file or repurpose it is Devlord's call, not this wiki's — but it should no longer be treated as an open live/legacy/parallel question.

**Devlord's direction for any future rebuild** (recorded in full on [[main]]): route outbound speech through [[cognitive_loop]]'s own gated decision cycle rather than reintroducing an independent, ungated loop like this file's own dormant one — so the agent can choose not to speak, not default to speaking whenever a threshold clears. Confirmed this pass: `cognitive_loop.py`'s `_decide()` already defaults to a "do nothing this cycle" outcome and already gates speech behind `language_desire` plus a cooldown, so the gating mechanism itself already exists; what's actually being proposed is making sure it stays the only path, rather than this file's own loop (or something like it) running alongside it ungated.

## Files Required

None — every interaction with `agent.brain`/`agent.memory`/`agent.emotion` is duck-typed, never imported.

## Files Used In

- [[cognitive_loop]] — one `try/except`-guarded call to `send_message()` inside `_execute_file_processing()`; confirmed functionally inert (packaging exclusion, per-process singleton isolation, and no registered connections — see [[main]] for the full reasoning).
- **Not** `main.py` — confirmed by direct read this pass. This file's own docstring's claim to be main.py-facing does not hold up in the current codebase.