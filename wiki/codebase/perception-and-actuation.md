---
type: system
status: ingested
---
# Perception & Actuation

💡 **What this is**: `ai_core/vision.py`, `audio_processors.py`, `actuators.py`, and `web_browser.py` — the raw sensor-in / command-out layer beneath [[observation-space]]'s assembled 128-dim vector. The headline finding: this layer is genuinely already built to run across all three of the design wiki's reality layers (Minecraft, Isaac Sim, physical robot), not just Minecraft — worth knowing given how much of the design wiki's physical-robot planning currently reads as aspirational.

## Vision — four capture backends, auto-detected

`vision.py`'s header states the same commitment as [[observation-space]]: _"Agents start with zero visual knowledge — no pretrained object labels, no assumed categories. Every meaningful visual pattern is discovered and named through experience."_

`VisionAdapter` auto-detects context and wires one of four `CaptureBackend` implementations:

- **`MinecraftVisionBackend`** — frame stream over the existing TCP/WebSocket connection (see [[communication-protocol]]).
- **`IsaacSimVisionBackend`** — reads a camera prim directly from the live Isaac Sim scene.
- **`RobotCameraBackend`** — V4L2 / GStreamer / any OpenCV-compatible device index. **This is the physical-robot camera backend, and it already exists in code** — not a placeholder, an actual implementation, ready to point at real hardware. Directly relevant to the design wiki's [[wiki/design/Robots/Creaking|Creaking]] and every other physically-embodied God Agent page.
- **`SyntheticBackend`** — deterministic gradient frames, so the pipeline always has something to run against in CI/testing with no camera attached at all.

Pipeline: raw frame → `FeatureExtractor` (a self-trained CNN, `_SimpleCNNExtractor`, not a pretrained one) → `OnlineVisualVocabulary` (k-means-style, `max_clusters=256` default, grows its own cluster vocabulary from experience — this is the agent's own visual "words," with no ImageNet labels anywhere in the loop). `push_minecraft_frame()` / `drain_frames()` are the actual entry/exit points [[communication-protocol]] calls.

## Audio — the same "route through reward, don't hardcode" discipline

`audio_processors.py`'s header is explicit about a specific anti-pattern it avoids: `process_audio_chunk()` used to hardcode emotion-detection heuristics directly onto the emotion system. Now, audio events route through `brain.evaluate_event()` (see [[brain-core]]) exactly like everything else — `detect_emotion()` still runs as a lightweight signal source, but its output becomes _payload data on a reward event_, not a direct emotion-system write. The cognitive loop, not the audio processor, decides when listening starts/stops (`brain.should_listen()`) — `AudioProcessor` is explicitly "a tool, not a decision maker," the same framing [[perception-and-actuation]]'s web-browsing section below uses for `WebBrowser`.

`AudioCapture` (16kHz default) → `SpeechRecognizer` + `AudioFeatureExtractor` → `AudioProcessor.process_audio_chunk()`. `MinecraftSoundAdapter` bridges in-game sound events specifically.

## Actuators — the exact wire format, and an Isaac Sim actuator too

`actuators.py` documents the literal TCP binary frame `ForgeIPCClient` sends to the Java `TCPServer` byte-for-byte:

```
[4]  agent_id length (uint32 BE)   [N]  agent_id (UTF-8)   [8]  tick (int64 BE, ms)
[4]  move_forward (f32)  [4]  move_strafe (f32)  [4]  yaw_delta (f32)  [4]  pitch_delta (f32)
[1]  action_flags (bit7=jump bit6=sneak bit5=attack bit4=use bit3=drop bit2=open_inv bit1=swap_hand bit0=sprint)
[1]  hotbar_slot (0xFF = no change)
[2]  god_ability name length (0 = none)  [M] god_ability (UTF-8)  [4][4][4] param1/2/3 (only if length > 0)
```

A fixed bug lived here: `send_action()` was packing one flags byte, but `TCPServer.java` was reading 7 individual boolean bytes — complete frame misalignment after the very first read, until the struct format on both sides was reconciled.

Backend classes: `MinecraftClient` / `ForgeIPCClient` (TCP) and `MinecraftWebSocketClient` (WS, port 11400 default — matches the `backend_port` scheme in [[architecture-overview]]) for the game layer, plus **`ActuatorAdapterIsaacSim`** — a real actuator backend for Isaac Sim, complete with `_DOFProfile` (degrees-of-freedom: name, index, drive mode, joint limits, revolute flag) for driving an articulated body. `ActuatorManager(backend: str, **kwargs)` is the single factory that picks between them, so the rest of the agent code never has to know which reality layer it's actually driving. See [[isaac-sim-integration]] for a similarly-named `IsaacSimActuator` — open question, not yet resolved, whether these are the same bridge under two names, two different layers, or genuine duplication.

## Web browsing — Playwright-backed, not scraping

`web_browser.py`: real browser automation (Playwright), inspired by VS Code 1.110's native browser integration — DOM interaction (click/type/scroll/hover), screenshots, console log capture, accessibility-tree snapshots for token-efficient page understanding. `WebBrowser` is explicitly "a TOOL only... [[brain-core]]'s `should_browse()` owns the decision of WHEN to browse" — the same tool/decider split as audio. Not autonomous by default: the agent decides most of the time, but an explicit user request ("check this", "look at that") queues a URL immediately rather than waiting for the agent's own `should_browse()` gate.

## What this confirms about the design wiki

The Minecraft-only framing implicit in a lot of the design wiki's Layer 1/Layer 2/Layer 3 discussion undersells what's actually built: `RobotCameraBackend` and `ActuatorAdapterIsaacSim` mean the perception and actuation layers are already reality-agnostic in code, not just in concept. What's actually missing for a physical robot isn't this layer — it's the physical hardware itself, and the `MinecraftClient`-equivalent glue code that would let a Raspberry Pi's sensors and motors plug into the same `CaptureBackend`/`ActuatorManager` abstractions Minecraft and Isaac Sim already use.