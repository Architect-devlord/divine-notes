---
type: file
status: ingested
---
# audio_processors.py

💡 **Role**: two independent audio sources feeding the same downstream contract — a real microphone pipeline and a Minecraft in-game-sound pipeline — both converging on `brain.evaluate_event()`. [[perception-and-actuation]] already covers the "route through reward, don't hardcode" discipline this file is built around in real depth; this page is the mechanical catalog plus the sound-tagging data and the danger/distance math the topic page doesn't spell out.

## Imports

`numpy`, `logging`, `typing`, `collections.deque`, `time` — external/stdlib. `pyaudio`, `speech_recognition`, `librosa` are all soft-imported with availability flags (`PYAUDIO_AVAILABLE`, `SPEECH_RECOGNITION_AVAILABLE`, `LIBROSA_AVAILABLE`) gating each capability independently — a missing `librosa` degrades feature quality, a missing `pyaudio`/`speech_recognition` disables the microphone path entirely, but neither takes down the Minecraft sound path. No internal project imports.

## Classes

- **`AudioCapture`** — raw PyAudio microphone capture, 16kHz/1024-sample chunks by default, a rolling ~6-second circular buffer (`deque(maxlen=100)`) fed by a callback running on PyAudio's own thread. `get_audio_chunk(duration)` concatenates however many recent chunks cover the requested duration.
- **`SpeechRecognizer`** — thin wrapper over Google's speech-recognition API (online, not offline). Does ambient-noise calibration exactly once (`noise_adjusted` flag), on the first real transcription attempt rather than at construction — avoids a calibration delay for agents that never actually end up listening.
- **`AudioFeatureExtractor`** — `extract_features()` degrades to two plain amplitude statistics (volume, peak) if `librosa` is absent, otherwise adds zero-crossing rate, spectral centroid, mean pitch (via `librosa.piptrack`, only over frames where pitch was actually detected), and tempo. `detect_emotion()` is a simple volume/pitch/tempo decision tree (loud+high-pitched+fast → excited, loud+high-pitched+slow → angry, quiet+low-pitched+slow → sad, quiet+low-pitched+fast → calm, else neutral) — explicitly a signal source for a reward-event payload, never a direct emotion write (see [[perception-and-actuation]]).
- **`AudioProcessor`** — the microphone-side integration class. `process_audio_chunk(duration=3.0)`: gated by a 3-second cooldown (`transcription_cooldown`) independent of the cognitive loop's own listening cadence; only attempts transcription at all if extracted volume exceeds 0.05 (a cheap gate before paying for a network round-trip to Google); builds an `audio_input` event tagged `speech` or `ambient` depending on whether anything was actually transcribed, and only stores a memory event if there was a transcription or a non-neutral emotion label — silence with nothing interesting about it isn't remembered. `start_listening()`/`stop_listening()` are the two methods [[cognitive_loop]]'s `brain.should_listen()` gate actually calls.
- **`MinecraftSoundAdapter`** — converts a JSON sound event (`sound_id`/`volume`/`distance`/`category`/`position`) from the mod's WebSocket into the exact same result shape `AudioProcessor.process_audio_chunk()` produces, so the cognitive loop handles both identically. `_get_tags()` matches `sound_id` against a 20-entry prefix table (`_MC_SOUND_TAGS`, module-level — creeper/skeleton/zombie/spider/enderman/wither/dragon/warden → danger+mob(+boss); explosions/combat/villager speech/experience orbs/note blocks/music each tagged distinctly), falling back to a smaller category-based map if no prefix matches. `_is_danger()` checks category `== 'hostile'` OR a 8-entry danger-prefix list (`_DANGER_PREFIXES`) — a slightly different, narrower set than the tag table's full "danger" tagging (e.g. skeleton/zombie/spider/enderman are tagged `danger` but aren't in the _danger-prefix_ list, so they don't independently spike urgency the way creeper/wither/dragon/warden/explosions/lava do). `receive_sound_event()` attenuates perceived volume by an inverse-square-ish distance falloff (`volume / max(1, (distance/5)^2)`) before anything else runs, and drops the event entirely below a 0.05 perceived-volume floor — a creeper heard from across the map never reaches the brain at all.

No async methods anywhere in this file — everything is synchronous, including the Minecraft sound path (called directly from whatever handler receives the WebSocket JSON).

## Functions

- `add_audio_processing_to_agent(agent)` — constructs both pipelines; the microphone side is wrapped in its own `try`/`except` since hardware may not exist in the runtime environment, while `MinecraftSoundAdapter` is constructed unconditionally (no hardware dependency). Exposes `agent.receive_minecraft_sound(event)` as the entry point the mod's WebSocket handler actually calls.

## Problems (faced by traditional AI systems / LLMs)

Audio-reactive systems commonly hardcode a direct mapping from an acoustic signal to an internal state change (loud sound → fear, detected laughter → joy) — which bypasses whatever personality or context-sensitivity the rest of the system has, and can't be tuned or overridden without touching the audio code itself. A narrower, Minecraft-specific problem: without any distance/volume modeling, every in-game sound event the mod forwards would need to already be pre-filtered server-side, coupling perception logic to the game client instead of keeping it in one place.

## Solutions

Both pipelines produce a payload (`emotion_label`, raw features) and hand it to `brain.evaluate_event()` — a `RewardSystem` interprets it through the agent's own personality weights exactly like any other event, meaning the same acoustic signal can affect two differently-tempered agents differently, and any future change to how sound _feels_ only has to happen in [[reward-and-learning-stack]], not here. Distance attenuation is computed once, in Python, from the raw distance the mod already sends — no server-side filtering needed, and the exact falloff curve can be tuned without touching the mod at all.

## Files Required

None from within this codebase — this file only reads `agent.brain`/`agent.memory` via duck typing, never imports their classes.

## Files Used In

- `agent.py` — `add_audio_processing_to_agent()` is called from `NPCAgent.__init__` (see [[agent]] and [[agent-runtime]]).
- `cognitive_loop.py` — `_think()` calls `should_listen()`-gated transcription and routes both speech and sound results into the perception/thought pipeline (see [[cognitive_loop]]).
- `communication_protocol.py` — calls `agent.receive_minecraft_sound()` when a `sound_event` arrives over `/ws/agent`; not yet given its own file-level page.
- `brain_core.py` — both pipelines' terminal call is `brain.evaluate_event()` (see [[brain_core]]).