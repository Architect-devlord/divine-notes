---
type: file
status: ingested
---
# Communication Protocol

💡 **What this is**: `ai_core/communication_protocol.py` — the actual wire format between a Minecraft client (`DWClientBot`) and its agent's Python process. See [[architecture-overview]] for where this sits in the bigger picture.

## Transport

WebSocket, chosen specifically for bidirectional low-latency communication — binary frames for images (JPEG-compressed) and MessagePack for action serialization, rather than JSON-over-WS for everything. Stated target: **<50ms round-trip, perception → action.** This is the number [[wiki/design/Robots|Robots]]'s Compute Architecture section borrows when setting a target for the physical-robot brain-server fallback option.

`run_tcp_action_loop(agent, agent_id, loop_hz=20.0)` is the documented fallback for when the WebSocket connection drops — a 20Hz polling loop, explicitly not the primary path. [[cognitive-loop]] and [[agent-runtime]] both start this alongside the WebSocket-based `CognitiveLoop`, not instead of it.

## Vision integration — no pretrained labels, by design

Frames arriving over this protocol get pushed directly into `agent.vision.push_minecraft_frame(frame_bgr)`. From there (`vision.py`, not deep-read beyond this entry point):

1. Features extracted via the agent's _own_ learned CNN — not a pretrained one.
2. A visual token assigned via online k-means clustering — the agent builds its own visual vocabulary from what it's actually seen, rather than being handed ImageNet-style category labels.
3. Frame stored in memory.
4. A proprioceptive vector fed to the WorldModel.

## Entity convention — same "no free labels" philosophy applied to language

Entities are transmitted as `(type_id, name, distance, angle)` tuples. `type_id` is just an integer the agent has to learn to associate with behavior over time — it's not told what it means. `name` is a raw string label from the Minecraft mod side, but the agent isn't told what _that_ means either; language grounding happens entirely separately, inside `brain_language.py`. The protocol deliberately hands the agent raw signal, not pre-interpreted meaning, consistently across both vision and entity perception.

## Port scheme

See [[architecture-overview]] for the full TCP/WS port-pairing convention (`tcp_port` from `agents.json`, `backend_port = tcp_port + 10000`).

## Three separate WebSocket endpoints, not one

`/ws/agent` (the main perception↔action loop), `/ws/sound` (structured JSON sound events specifically — no audio hardware needed, headless/cloud-friendly, carries semantic metadata raw PCM can't), and the raw-audio path _inside_ `/ws/agent` itself for in-game voice chat. The file's own docstring is explicit that the sound-event and raw-audio paths "coexist" deliberately rather than one superseding the other.

## Two real bugs worth knowing about, found reading the file in full

**A serious audio bug**: raw Minecraft voice-chat audio arriving over `/ws/agent` used to get decoded correctly into a normalized float32 array, then never actually used — all three downstream consumers (buffering, volume-gated transcription trigger, speech recognition) re-decoded the raw bytes themselves, receiving int16-range values where float32-range values were expected. Concretely: the volume check was permanently true regardless of actual volume, and transcription's own internal ×32767 rescale compounded into an overflow. Fixed by decoding once and passing that single value everywhere. See [[wiki/codebase/files/communication_protocol|communication_protocol.py]] for the full account.

**The GRPO duplication question, now resolved**: an earlier ingest pass ([[self_supervised_trainer]]) flagged a comment suggesting `communication_protocol.py` might still have a second, separate GRPO implementation. Confirmed false — `_grpo_policy_update()` here is a thin, guarded call into the one verified `grpo_update()` implementation (monkeypatched onto the live policy from `self_supervised_trainer.py`), not a duplicate. It's called from two trigger sites (here, and [[cognitive-loop]]'s continual-learning worker) with an explicit guard so only one fires per signal — a genuinely intentional two-consumer/one-owner design, not accidental duplication.

## Not yet given its own deep pass

`main.py`'s actual FastAPI route registrations for `/ws/agent`/`/ws/sound` (presumed to exist, not directly confirmed), and `packager.py`/`chat_system.py`/`god_controls.py`/`breeding_system.py`/`web_browser.py` are all still topic-level-only or entirely unread.