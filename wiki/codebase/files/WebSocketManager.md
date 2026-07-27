---
type: file
status: ingested
---
# WebSocketManager.java

💡 **Role**: the actual live wire to Python — connects to `{url}:{port}/ws/agent` (confirmed matching [[agent]]'s own documented route exactly), captures and sends perception frames at 20 FPS, and decodes inbound action frames back into game input. This is the file [[communication-protocol]]'s biggest remaining gap pointed at, and reading it directly resolves most of that page's open items — a genuinely rich file, its own class docstring already documenting nine named fixes (F1–F4, "original"; W1–W5, "new") with real precision.

## Imports

Minecraft/Forge client, entity, level, and physics types (`Minecraft`, `Frustum`, `BlockPos`, `Entity`/`Animal`/`EnderDragon`/`WitherBoss`/`ItemEntity`/`Monster`/`Player`/`Projectile`, `ClipContext`, `Level`, `BlockState`, `AABB`, `HitResult`, `Vec3`). `com.divineworld.client` (`DWClientMod`, `entity.GodEntityManager`, `vision.AudioCaptureSystem`, `vision.VisionCaptureSystem`, `control.ActionExecutor`) — internal. `java.net.http` (`HttpClient`, `WebSocket`), `java.io.ByteArrayOutputStream`, `java.nio.ByteBuffer`, `java.util.concurrent.*` — stdlib.

## The nine documented fixes, condensed

- **F1–F4** (predate this rewrite): `wsFuture.join()` off the main thread; JPEG encoding off the main thread; `sendBinary()` off the main thread; reduced default capture resolution.
- **W1 — audio metadata wire mismatch**: Java used to always write `sampleRate`/`channels`/`bitsPerSample` after the audio section, even with zero audio bytes; Python's decoder only reads those fields when `audio_len > 0`. A silent agent left 6 unconsumed bytes that got misread as the next field. Fixed by only writing that metadata when audio is actually present, confirmed in `buildPerceptionFrame()`'s own wire-layout comment below.
- **W2 — `entity_count` hardcoded to 0**: perception's entity list was always empty — the agent could never see a zombie, player, or animal through its own perception frame, no matter how close. Fixed by `collectNearbyEntities()` (below), a genuinely two-phase visual/audible system.
- **W3 — sound events never sent**: Python's decoder reads a `sound_count` field Java never wrote; the gap was silently swallowed by a Python `try/except`. Fixed with a static sound-event queue other client classes feed via `queueSoundEvent()`.
- **W4 — thread leak on reconnect**: `scheduleReconnect()` used to reassign `perceptionExecutor` to a fresh instance without shutting the old one down first — one leaked daemon thread per disconnect. Fixed with an explicit `shutdownNow()` before replacement.
- **W5 — reconnect could get permanently stuck**: if the very first connection attempt failed before `onOpen()` ever ran, `perceptionExecutor` was still `null`, and `scheduleReconnect()`'s null-check meant it silently gave up rather than retrying. Fixed with a dedicated `reconnectExecutor`, created once, never reassigned.

## Perception: capture and send

- `initialize(url, port, agentId)` — connects async (non-blocking), registers a `WebSocket.Listener` handling `onOpen` (sends a JSON handshake, starts the perception loop), `onBinary` (accumulates fragmented chunks into `msgAccum` until `last=true`, then peeks the 8-byte MAGIC+frame-type header to route to `handleChatFrame`/`handleActionFrame`), `onClose`/`onError`/the connect future's `.exceptionally()` (all three call `scheduleReconnect`).
- `startPerceptionLoop()` — a `scheduleAtFixedRate` on its own executor, 200ms initial delay then every 50ms (20 FPS), hopping onto the Minecraft main thread each tick to call `captureAndScheduleEncode()`.
- `captureAndScheduleEncode()` — reads cheap player state and does the GPU pixel readback + entity/block-neighbourhood scan on the main thread (required), then hands JPEG encoding and the actual `sendBinary()` call off to a dedicated single-thread `encodeExecutor` — the F2/F3 fixes in concrete form.
- `collectNearbyEntities(mc)` — genuinely two-phase: **Phase 1 (visual)** uses `entitiesForRendering()` (already frustum-culled) plus an explicit `ClipContext` line-of-sight raycast from the player's eye to each candidate's centre, excluding anything occluded by a solid block; these get their real `type_id` and registry name. **Phase 2 (audible)** — anything within a smaller `SOUND_RADIUS` that didn't pass Phase 1 gets sent as `ENTITY_UNKNOWN` with an empty name: the agent senses _something_ nearby without knowing what, mirroring hearing before seeing. Each entity's egocentric forward/right/vertical offsets and movement speed are also computed and included (an additive extension beyond the original W2 fix's basic fields, per an in-line "Step 1" comment) — audible-only entities get `movementSpeed` forced to `0`, since speed can't be inferred from sound alone.
- `buildPerceptionFrame(...)` — the actual wire construction, with its own precise layout comment (`MAGIC`, frame type, agent ID, timestamp, image data + dimensions, health/hunger/position/yaw/pitch, entity count + per-entity blobs, audio length + data + conditional metadata (W1's fix), sound-event count + per-event JSON blobs (W3's fix)) — explicitly cross-referenced against `BinaryProtocol.unpack_perception` on the Python side.

## Inbound: action and chat frames

- `handleChatFrame(buf)` — reads a length-prefixed UTF-8 string and calls `ClientPacketListener.sendChat()` (truncated to 256 chars) — this is the actual mechanism by which an agent's spoken decision reaches real Minecraft chat.
- `handleActionFrame(buf)` — decodes `moveForward`/`moveStrafe`/`yawDelta`/`pitchDelta`/`actionFlags`/`hotbarSlot`, then an optional short-length-prefixed god-ability name string plus three trailing floats (`p1`/`p2`/`p3`) if present. Dispatches movement/flags to [[ActionExecutor]]`.executeAction()` and, if an ability name is present, to `GodEntityManager.executeGodAbility(name, p1, p2, p3)`.
- **Confirmed dead, 2026-07-16 — escalated from a genuinely open thread to a verified finding**: `GodEntityManager.executeGodAbility()` (checked directly, briefly, when this page was first written) is client-visual-only — it calls `IGodEntity.useAbility()` on whatever god entity is already in the world. A full read of [[GodEntityManager]] since then found something stronger: it never actually reaches that call at all. `currentGodEntity`, the field `executeGodAbility()`'s only guard condition depends on, is never assigned anywhere in the entire repository — confirmed by an exhaustive search, not a guess. So this dispatch is not merely "visual, not server-authoritative" — it's completely inert, for any ability, from any god, currently. See [[GodEntityManager]] for the full account and [[god-abilities]] for how this fits the broader, already-resolved server-side routing question.

## Outbound: proximity chat and lifecycle

- `sendChatObservation(speaker, message)` — sends `{"type": "chat_heard", "agent_id", "speaker", "message", "timestamp"}` as a plain JSON **text** WebSocket message (not a binary frame) over this same connection. **Checked directly against [[ProximityChatHandler]], corrected**: this page previously guessed this method was "almost certainly" what that handler calls for NPC-specific chat delivery. It isn't — [[ProximityChatHandler]] never calls this method at all; it uses `PythonBackendClient.notifyChatHeard()` (HTTP) uniformly for every agent type, NPC and god alike, with NPCs additionally receiving chat via vanilla's `ClientChatReceivedEvent` on their own client. Where this method actually gets called from, if anywhere, is unconfirmed — a genuinely open thread now, not a wrong guess left standing.
- `shutdown()` — stops the perception executor, closes the socket, cleans up [[VisionCaptureSystem]].

## Problems (faced by traditional AI systems / LLMs)

Two recurring themes across the nine fixes, both general to any bidirectional binary protocol between two independently-evolving codebases: a struct layout with a conditional section (audio metadata only present when audio exists) is exactly the kind of thing that silently desyncs if one side's writer and the other side's reader aren't updated in lockstep — and resource lifecycle bugs (the two thread-leak/stuck-reconnect fixes) that only show up under a failure condition (a dropped connection) that's easy to under-test compared to the happy path.

## Solutions

Fixing W1 by making the writer's behavior match the reader's actual assumption, rather than changing the reader to tolerate both cases, is the more surgical fix — it keeps one side as the single source of truth for the format rather than making both sides more permissive. W4/W5 are solved by the same underlying principle applied twice: a resource that might need replacing (an executor) should either be explicitly torn down before reassignment, or never reassigned at all if reassignment isn't actually necessary — the dedicated, permanent `reconnectExecutor` is the second flavor of that fix.

## Files Required

- [[ActionExecutor]] — `executeAction()`, movement/flags dispatch.
- `GodEntityManager.java` (client-side, briefly checked, not yet given its own full page) — `executeGodAbility()`, confirmed client-visual-only.
- [[VisionCaptureSystem]] / [[AudioCaptureSystem]] — pixel and audio capture, not yet given their own pages this pass.
- `com.divineworld.client.DWClientMod` — logger access.

## Files Used In

- Constructed/initialized presumably from `DWClientMod`'s own startup sequence — not yet confirmed directly.
- **Not** [[ProximityChatHandler]], confirmed by direct read of that file — see the correction above. That file's own chat-delivery path never touches this file's `sendChatObservation()` at all.