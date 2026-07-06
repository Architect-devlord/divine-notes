---
type: system
status: ingested
---

# DWClientBot: Sensing, Control & Networking Plumbing

💡 **What this is**: the Minecraft-client-side machinery that turns the game into sensor data going out and actions coming in — [[perception-and-actuation]] covers the Python side (`vision.py`, `audio_processors.py`, `actuators.py`) that this actually feeds. This page is the Java capture/execution layer underneath that, plus the small packet system that keeps [[forms-and-disguise]]'s form state in sync across clients.

## `VisionCaptureSystem` — three fixes for old hardware, all in one class

The class header documents its own history as a rewrite, not an original design — worth knowing if it gets touched again, since the fixes are load-bearing, not incidental:

- **Split readback from encoding.** The old single-method version did GPU screenshot readback, pixel-scaling, and JPEG compression all on the main thread — 100–500ms of blocking on slow hardware, a visible freeze every perception tick. Now `grabPixels()` (GPU readback + scale, must stay on the main thread) and `encodePixelsToJPEG()` (compression, safe off-thread) are separate methods; [[perception-and-actuation]]'s Vision Adapter architecture is what calls the first on the main thread and schedules the second on an encode executor.
- **Default resolution halved**, 640×480 → 320×240 (override via `-Ddw.vision.width/height`) — a quarter of the pixels to process per frame.
- **Direct ABGR unpacking** into a reused `int[]` scratch buffer (one allocation at startup, not per frame) instead of a per-pixel `getPixelRGBA()` call, avoiding both the slower accessor and repeated GC pressure.
- **Explicit JPEG quality** via `ImageWriter`/`ImageWriteParam` rather than `ImageIO.write()`'s default (~0.75) — defaults to 0.5 (override via `-Ddw.vision.quality`), trading a bit of visual fidelity for smaller frames and faster sends.

Capture is capped at 20 FPS (`CAPTURE_INTERVAL_MS = 50`), matching the frame rate [[perception-and-actuation]] documents on the Python side.

## `AudioCaptureSystem` — loopback capture, not microphone capture

Captures what Minecraft itself is *playing* (footsteps, mobs, ambient, music) rather than a real microphone: opens a secondary OpenAL loopback device via `ALC_EXT_capture` to record exactly what's already been mixed for playback, falling back to `javax.sound.sampled`'s `TargetDataLine` on the system's default loopback monitor if ALC capture isn't available (some Linux drivers). Fixed output format: 16-bit signed mono PCM at 22,050Hz — the same rate [[perception-and-actuation]]'s `AudioCapture` documents on the Python side, so no resampling is needed between the two layers.

## `ActionExecutor` — one bit-packed byte, one action frame

`executeAction()` unpacks a single flags byte (bit 7→0: jump, sneak, attack, use, drop, open_inv, swap_hand, sprint — matching [[human-controller-debug]]'s `_pack_action` wire format exactly, since both the AI agent's own action pipeline and the human-controller debug tool ultimately drive the same client-side entry point) plus movement/camera floats, and applies them via Forge's `KeyMapping.setDown()` — the same mechanism a real keypress would trigger, so the rest of the client (movement prediction, animation) behaves identically either way. Attack/use share a 4-tick cooldown; opening the inventory has its own 20-tick cooldown, both simple `int` counters decremented once per call.

Inventory slot clicks are a separate, string-based sub-protocol layered on top: `"inv:SLOT,BUTTON,CLICK_TYPE"` (slot per vanilla's `InventoryMenu` layout — 9-35 main inventory, 36-44 hotbar, 45 offhand; click-type ordinals matching vanilla's `ClickType` enum) routes to `MultiPlayerGameMode.handleInventoryMouseClick()` — the identical call vanilla makes for a real mouse click, so it's fully server-authoritative and gets corrected automatically if client-side prediction was wrong. `"screen:close"`/`"screen:inv"` round out the set for opening/closing GUIs. Interactive blocks (crafting tables, furnaces) need no special handling beyond this — setting `use=true` while looking at one triggers the same vanilla open-screen packet flow a player walking up and right-clicking would.

## Morph sync — one packet, two independent classes

`MorphSyncPacket` (server, in `com.divineworld.network`) and `ClientMorphSyncPacket` (client, in `com.divineworld.client.network`) are deliberately two separate classes with matching wire formats (`UUID` + two length-capped strings) rather than one shared class — the two mods have zero cross-imports by design, so each side owns its own copy. `ClientNetworkHandler` registers the client class at packet ID 0 on the `"divineworld:main"` channel, protocol version `"2"`, which must match the server mod's own registration exactly or the connection silently drops. On receipt, the client class just pushes into `MorphStateCache` for [[forms-and-disguise]]'s `TransformationHandler` to drain on the next tick — no logic of its own beyond that hand-off. (`ClientMorphSyncPacket`'s own header comment claims "packet ID 1" — the actual registration code assigns ID 0, since `ClientNetworkHandler` only ever registers this one packet type. Minor stale comment, not a functional issue.)
