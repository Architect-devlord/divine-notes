---
type: system
status: ingested
---

# Communication Protocol

💡 **What this is**: `ai_core/communication_protocol.py` — the actual wire format between a Minecraft client (`DWClientBot`) and its agent's Python process. See [[architecture-overview]] for where this sits in the bigger picture.

## Transport

WebSocket, chosen specifically for bidirectional low-latency communication — binary frames for images (JPEG-compressed) and MessagePack for action serialization, rather than JSON-over-WS for everything. Stated target: **<50ms round-trip, perception → action.** This is the number [[wiki/design/Robots|Robots]]'s Compute Architecture section borrows when setting a target for the physical-robot brain-server fallback option.

`run_tcp_action_loop(agent, agent_id, loop_hz=20.0)` is the documented fallback for when the WebSocket connection drops — a 20Hz polling loop, explicitly not the primary path. [[cognitive-loop]] and [[agent-runtime]] both start this alongside the WebSocket-based `CognitiveLoop`, not instead of it.

## Vision integration — no pretrained labels, by design

Frames arriving over this protocol get pushed directly into `agent.vision.push_minecraft_frame(frame_bgr)`. From there (`vision.py`, not deep-read beyond this entry point):

1. Features extracted via the agent's *own* learned CNN — not a pretrained one.
2. A visual token assigned via online k-means clustering — the agent builds its own visual vocabulary from what it's actually seen, rather than being handed ImageNet-style category labels.
3. Frame stored in memory.
4. A proprioceptive vector fed to the WorldModel.

## Entity convention — same "no free labels" philosophy applied to language

Entities are transmitted as `(type_id, name, distance, angle)` tuples. `type_id` is just an integer the agent has to learn to associate with behavior over time — it's not told what it means. `name` is a raw string label from the Minecraft mod side, but the agent isn't told what *that* means either; language grounding happens entirely separately, inside `brain_language.py`. The protocol deliberately hands the agent raw signal, not pre-interpreted meaning, consistently across both vision and entity perception.

## Port scheme

See [[architecture-overview]] for the full TCP/WS port-pairing convention (`tcp_port` from `agents.json`, `backend_port = tcp_port + 10000`).
