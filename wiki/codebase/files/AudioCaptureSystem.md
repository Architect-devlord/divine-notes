---
type: file
status: ingested
---
# AudioCaptureSystem.java

💡 **Role**: captures Minecraft's own rendered audio — footsteps, mob sounds, ambient effects, music — as raw PCM, so an agent can "hear" the game the way a player would. Two independent capture strategies, tried in order, not a single implementation with no fallback.

## Imports

`org.lwjgl.openal` (`AL10`, `ALC10`, `ALC11`) — LWJGL's OpenAL bindings. `javax.sound.sampled.*` — the JDK's own audio API, used only as a fallback. `java.nio.*`, `java.util.concurrent.atomic.AtomicBoolean` — stdlib. `com.divineworld.client.DWClientMod` — internal.

## Strategy

**Primary: ALC loopback capture.** Minecraft's audio all goes through OpenAL; `ALC11.alcCaptureOpenDevice(null, ...)` with a `null` device name opens the driver's _default capture device_, which on most systems is the loopback/monitor of whatever's currently playing — this records exactly what OpenAL has already mixed, without inserting anything into the normal playback path. **Fallback: `javax.sound.sampled.TargetDataLine`**, tried only if ALC capture is unavailable (the docstring specifically notes this happens on some Linux driver setups) — captures from the system-default loopback monitor instead, a different mechanism reaching for the same conceptual signal.

Output format is fixed and documented precisely: 16-bit signed PCM, mono, 22,050Hz, little-endian — confirmed to match exactly what [[WebSocketManager]]'s perception-frame builder writes as the conditional audio metadata fields (the W1 fix on that page: only written when audio bytes are actually present).

## Methods

- `initialize()` — tries `tryInitALC()` first; only calls `tryInitJavaxSound()` if that fails. Sets `running` based on whichever succeeded (or neither), logging a clear warning if audio capture ends up completely unavailable — perception frames still go out, just with silent audio, rather than the whole system failing.
- `tryInitALC()` — opens the capture device with a ring buffer sized at 4× one tick's worth of samples ("for safety" per the in-line comment — headroom against a slow consumer falling behind by a frame or two), starts capture, checks the ALC error state explicitly rather than trusting a non-null handle alone.
- `tryInitJavaxSound()` — builds a matching `AudioFormat`, checks `AudioSystem.isLineSupported()` before attempting to open the line (so an unsupported-format failure is a clean early return, not an exception from deeper in the JDK).
- `captureAudioFrame()` — the actual per-tick read, meant to be called from the same thread as vision capture (the main thread, per the javadoc). Dispatches to whichever backend is active; returns an empty array (never `null`) when capture is unavailable or nothing new has accumulated — a caller can treat "no audio" and "capture is broken" identically without a null check.
- `captureFromALC()` — asks ALC how many samples are actually ready (`ALC_CAPTURE_SAMPLES`) before reading, caps the read at one tick's worth even if more is available (excess stays buffered for next tick rather than being dropped or over-read), reads into the reusable scratch buffer, copies out only the actual bytes read.
- `captureFromJavaxSound()` — the same read-what's-available-capped-at-one-tick pattern, via `TargetDataLine.read()` instead.
- `getSampleRate()`/`getChannels()`/`getBitsPerSample()`/`isActive()` — fixed-value/status accessors.
- `cleanup()` — stops and closes whichever backend is active, each step independently guarded against throwing.

## Problems (faced by traditional AI systems / LLMs)

Not AI/ML-specific — the general problem is audio-loopback capture's own well-known portability gap: there's no single, universally-supported API for "record what's currently playing" across every OS/driver combination, so a robust implementation needs more than one strategy and a graceful way to have neither succeed without crashing the caller.

## Solutions

Two independent, ordered strategies with a shared output contract (both produce the same fixed PCM format) mean the rest of the system never needs to know which one is actually active — `captureAudioFrame()`'s callers get a consistent interface regardless. Capping every read at exactly one tick's worth, on both paths, keeps the perception loop's timing predictable even when the underlying audio buffer has accumulated more than expected.

## Files Required

None beyond LWJGL's OpenAL bindings and the JDK's own `javax.sound.sampled`.

## Files Used In

- [[VisionCaptureSystem]] — owns this class's lifecycle (`initialize()`/`cleanup()` both delegate here).
- [[WebSocketManager]] — `captureAudioFrame()`, called during perception-frame construction; `getSampleRate()`/`getChannels()`/`getBitsPerSample()` for the conditional metadata fields.