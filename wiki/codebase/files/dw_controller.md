---
type: file
status: ingested
---
# dw_controller.py

💡 **Role**: `ControllerRuntime` — permission-gated, real system-hardware sensor access (camera via OpenCV, microphone via `sounddevice`) feeding directly into an agent's memory and emotion systems. Distinct from [[web_browser]] and the Minecraft perception pipeline: this is the one place in the whole backend that reaches past the game and the browser into actual host-machine hardware. Its own module docstring is unusually direct about two things it's specifically _not_: it uses `agent.emotion.emotions[key] = value` (the dict `EmotionSystem` actually exposes) rather than a hypothetical `.add()` method, and `agent.memory.remember(event, tags=[...])`, the real `UnifiedMemoryStore` API — both stated as if written to head off a specific mistake, not generically.

Resolves both pointers carried forward from earlier this session as open threads. [[agent]]'s `_get_controller()` lazily constructs exactly one `ControllerRuntime(global_agent)` module-level singleton, confirmed matching this file's constructor. `/api/controller/detect-devices` is a genuine special case worth noting: it builds a `ControllerRuntime` via `__new__` rather than `__init__`, hand-setting only `.agent` before calling `list_cameras()`/`list_microphones()` — a deliberate way to probe available hardware without needing an active agent or going through the full permission-gated constructor first.

## Imports

`logging`, `threading`, `time`, `typing` — stdlib. `cv2` (OpenCV), `numpy` — external, both hard requirements (no guard, unlike most optional dependencies found elsewhere this ingest — if `cv2` isn't installed, importing this file fails outright). `sounddevice` (aliased `sd`) — the one dependency here that _is_ guarded, `try/except ImportError: sd = None`, checked before every actual use. `ai_core.agent.NPCAgent` — type hint only.

## Classes

### `CameraCapture`

Threaded OpenCV capture. `start()` opens the device, sets resolution, launches a daemon thread; `_loop()` reads frames at the configured FPS into a lock-protected `self._frame`; `get_frame()` returns a copy (never the live buffer, avoiding a caller mutating frame data mid-capture); `stop()` joins the thread and releases the capture handle.

### `MicrophoneCapture`

Threaded `sounddevice` input stream. Raises immediately in `__init__` if `sd is None`. `start()` registers a callback that appends each chunk to a capped ring buffer (`max_buffer_chunks`, oldest dropped first); `get_audio_chunk()` concatenates and clears the whole buffer in one call — a caller polling less often than audio arrives gets everything since the last poll, concatenated, not just the latest chunk.

### `ControllerRuntime`

- `__init__(agent, max_camera_checks=6, callback=None)` — `enabled_permissions` defaults every key (`camera`, `microphone`, `filesystem`, `network`) to `False` — nothing is available until explicitly granted. `stats` dict tracks frames/audio-chunks/files processed and total learning events.
- `list_cameras(limit=None)` / `auto_detect_camera(prefer_index=None)` — probes camera indices directly (opens, reads one frame, releases, with a `0.05s` sleep after each release specifically so the OS actually frees the handle before the next probe), returns those that respond; auto-detect prefers the given index if it's in the responding set, otherwise the highest-resolution camera found.
- `list_microphones()` — wraps `sd.query_devices()`, filtered to input-capable devices; returns `[]` (not an error) if `sounddevice` isn't installed.
- `start_camera()` / `start_microphone()` — both permission-gated first (`enabled_permissions.get(...)`, logging and returning `False` rather than raising if denied), then construct and start the matching capture class.
- `grant_permissions(permissions)` — the method `/api/controller/activate` calls after a user acknowledges a permission dialog in the frontend; only enables keys that already exist in `enabled_permissions`, silently ignoring anything else.
- `can_access_filesystem()` / `can_access_network()` / `can_use_camera()` / `can_use_microphone()` — plain permission reads. Worth noting precisely: `filesystem`/`network` permissions exist in the dict and have their own check methods, but nothing in this file (or found calling into it) ever actually gates a filesystem or network action behind them — the camera and microphone are the only two of the four permissions this file itself enforces anywhere.
- `learn_from_frame(frame)` / `learn_from_audio(audio, sample_rate)` — the actual sensor-to-agent bridge. Both: stash the raw data in `agent.perception_buffer` (created on first use if absent) for other modules to read, write a summarized (not raw-data) memory event tagged appropriately, and nudge specific emotions based on a simple signal-derived proxy — brightness-as-novelty for vision (`surprise`/`curiosity`), RMS-as-intensity for audio (`surprise`/`anticipation`) — both explicitly commented as simple proxies, not real novelty/salience scoring.
- `start_multimodal_learning(vision=True, audio=True)` — permission-checks and starts each requested modality independently (a denied or failed camera doesn't block audio from starting), spawns one polling thread per active modality (`_vision_loop`/`_audio_loop`, 0.1s/0.5s poll intervals respectively) that pulls from the capture buffer and calls the matching `learn_from_*` method.
- `stop()` — joins both threads, stops both capture objects if present.
- `get_stats()` — merges the running counters with live `camera_active`/`microphone_active`/`memory_size` reads.
- `__enter__`/`__exit__` — context-manager support, `__exit__` just calls `stop()`.

## Problems (faced by traditional AI systems / LLMs)

Genuinely general to any system given real hardware access on someone's behalf, not AI/ML-specific: sensor access needs to be off by default and explicitly, individually grantable, and a permission denial or a hardware failure in one modality shouldn't take down a different, unrelated one that's working fine.

## Solutions

The `enabled_permissions` dict defaulting everything to `False`, checked at the point of use rather than trusted from a caller, is a plain, correct answer to the first half. Independent try/except-wrapped start paths per modality inside `start_multimodal_learning()` — a camera failure sets `vision = False` and logs, without touching the audio branch at all — is a plain, correct answer to the second half. Neither is a sophisticated pattern, but both are the right shape for what the problem actually needs.

## Files Required

- `ai_core/agent.py` — `NPCAgent`, type hint only. See [[agent]].

## Files Used In

- [[agent]] — `_get_controller()` lazily constructs the one module-level `ControllerRuntime` singleton; `/api/controller/detect-devices`, `/api/controller/activate`, `/api/controller/deactivate`, `/api/controller/status` are its four routes. Worth a small, low-stakes note: `/api/controller/status`'s response construction uses `hasattr(ctrl, 'running')`/`hasattr(ctrl, 'enabled_permissions')` guards — checked directly, `ControllerRuntime.__init__` always sets both unconditionally, so these are defensive against an interface that (confirmed) always has them, the same minor pattern already documented at length for [[chat_system]] elsewhere in this ingest, here in a much smaller, lower-consequence form.