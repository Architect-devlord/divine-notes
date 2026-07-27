---
type: file
status: ingested
---
# VisionCaptureSystem.java

💡 **Role**: screen capture for perception — GPU pixel readback and JPEG encoding, deliberately split across two methods so [[WebSocketManager]] can run one on the main thread and the other off it. A clean, three-fix file (F2a/F2b/F2c, plus F4), all named and explained in its own class docstring with real, specific numbers rather than vague claims.

## Imports

`com.mojang.blaze3d.platform.NativeImage`, `net.minecraft.client.Minecraft`/`Screenshot` — Minecraft. `javax.imageio.*`, `java.awt.image.BufferedImage`, `java.io.ByteArrayOutputStream` — stdlib. `com.divineworld.client.DWClientMod` — internal.

## The documented fixes

- **F2a — pixel readback split from JPEG encoding.** The docstring gives real numbers for why this mattered: the old single-method version did GPU readback, a 307,200-iteration scale loop, a second 307,200-iteration convert loop, and JPEG compression, all on the main thread — 100–500ms on old hardware, a visible freeze every 50ms at the 20 FPS perception rate. Split into `grabPixels()` (GPU readback + scale, main-thread-only) and `encodePixelsToJPEG()` (compression, safe from any thread) — exactly the two methods [[WebSocketManager]]`.captureAndScheduleEncode()` calls on the main thread and its `encodeExecutor` respectively.
- **F4 — default resolution halved**, 640×480 → 320×240 (a quarter the pixels), overridable via `-Ddw.vision.width`/`-Ddw.vision.height` system properties.
- **F2b — direct `NativeImage` unpacking with a reusable scratch buffer.** `NativeImage` already stores pixels as packed ABGR ints; reading one int and unpacking with bit shifts, into a pre-allocated `int[]` reused every frame, replaces a slower per-pixel `getPixelRGBA()` call pattern and avoids garbage-collector pressure from allocating a fresh array every frame.
- **F2c — explicit JPEG quality via `ImageWriter`**, rather than `ImageIO.write()`'s shortcut (which uses a fixed ~0.75 quality with no way to change it) — default `0.5`, overridable via `-Ddw.vision.quality`.

## Methods

- `initialize()` — reads the three system-property overrides, allocates the pixel scratch buffer once, and calls `AudioCaptureSystem.initialize()` — vision owns audio's lifecycle, not a peer relationship.
- `grabPixels()` — rate-limited internally to 20 FPS (returns `null` if called too soon, independent of whatever caller cadence exists above it); `Screenshot.takeScreenshot()` for the GPU→CPU readback, scaled and unpacked into the scratch buffer in one pass, then **copied** into a fresh array before returning — the copy is deliberate, so the encode thread reads a consistent snapshot even if a new capture starts before encoding the previous one finishes.
- `encodePixelsToJPEG(pixels, w, h)` — wraps the int array directly into a `BufferedImage` (no extra copy), then the explicit-quality `ImageWriter` path from F2c.
- `captureScreenAsJPEG()` — `@Deprecated`, kept only for any caller still using the old combined signature; just calls the two split methods back to back.
- `getWidth()`/`getHeight()`/`getQuality()` — the current, possibly-overridden values.
- `cleanup()` — clears the scratch buffer and delegates to `AudioCaptureSystem.cleanup()`.

## Problems (faced by traditional AI systems / LLMs)

Not AI/ML-specific — the interesting problem here is a general real-time-rendering one: any per-frame CPU work that competes with a fixed-rate game loop has to either be fast enough to fit in the frame budget or be moved off the thread that owns the budget, and a naive single-method implementation that mixes a hard main-thread requirement (GPU readback) with heavy, thread-agnostic CPU work (compression) forces everything onto the expensive thread by construction.

## Solutions

Splitting the method in two, at exactly the boundary between "must be main thread" and "can be anywhere," is the correct fix — not a workaround, a removal of the false constraint. The reusable scratch buffer (F2b) and explicit quality control (F2c) are smaller, complementary wins on top of that split: less garbage-collector pressure, and a deliberate quality/size tradeoff instead of an implicit one.

## Files Required

- [[AudioCaptureSystem]] — lifecycle-owned (`initialize()`/`cleanup()` both delegate).

## Files Used In

- [[WebSocketManager]] — `grabPixels()` on the main thread, `encodePixelsToJPEG()` on the encode executor, exactly matching the split this file was built for.