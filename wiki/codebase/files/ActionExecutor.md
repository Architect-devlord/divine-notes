---
type: file
status: ingested
---
# ActionExecutor.java

💡 **Role**: translates a decoded action into actual Minecraft client input — movement, camera, attack/use/drop/inventory-toggle/sprint, plus two more specialized paths (`inv:`-prefixed inventory-slot actions, and chat) that [[TCPServer]] calls directly rather than going through the normal movement path. **Does not handle god abilities** — confirmed by a full read, not inferred from absence. That's [[WebSocketManager]]'s own dispatch to `GodEntityManager` instead; see that page's own flagged open thread about how the ability trigger actually reaches the server.

## Imports

Minecraft/Forge client and input types (`Minecraft`, `KeyMapping`, `InteractionHand`, `ClickType`, `Slot`). `com.divineworld.client.DWClientMod` — internal.

## Methods

- `executeAction(controls)` — the normal path, called from [[WebSocketManager]]`.handleActionFrame()`. Movement: sets `KeyMapping` pressed-state directly for forward/back/left/right/sprint/jump based on sign and threshold checks on `moveForward`/`moveStrafe` and the sprint/jump bits in `actionFlags`, rather than synthesizing raw key events — the mod is driving the same input layer a real keyboard would, not injecting at a lower level. Camera: applies `yawDelta`/`pitchDelta` directly to the player's rotation, pitch clamped to `[-90, 90]`. Attack/use: single-tick `leftClick`/`rightClick` triggers via `Minecraft`'s own click-handling methods (not held down), gated by their respective `actionFlags` bits. Hotbar slot, drop, inventory-open, swap-hand: straightforward flag/value reads mapped to their matching vanilla actions.
- `executeInventoryAction(abilityString)` — parses a `"inv:SLOT,BUTTON,CLICK_TYPE"` string (the format `TCPServer`, not this class, is responsible for recognizing and routing here — this method's own docstring says as much) and calls `mc.gameMode.handleInventoryMouseClick()` with the parsed values, which sends the real `ServerboundContainerClickPacket` through vanilla's own networking — this class never builds a packet itself for this path either, it just calls the existing Minecraft API that does.
- `executeScreenAction(abilityString)` — `"screen:close"` calls `mc.setScreen(null)` (also sends vanilla's own close-container packet as a side effect); `"screen:inv"` opens the player's own inventory screen locally, no packet at all since opening your own inventory is client-only state.
- `executeChatAction(message)` — sends a chat message via `ClientPacketListener.sendChat()`. Its own docstring is explicit about why this exists here at all, duplicating something [[WebSocketManager]] already does in `handleChatFrame()`: [[TCPServer]]'s own action-frame parsing didn't have any way to deliver a trailing chat section before this method existed, so it's not a redundant reimplementation so much as a gap-filler for a code path [[WebSocketManager]] doesn't share.

## Problems (faced by traditional AI systems / LLMs)

Not AI/ML-specific — the interesting problem this file solves is a plain input-simulation one: making an automated controller's actions indistinguishable, at the API level, from a human's — setting the same `KeyMapping` state and calling the same click/rotation methods a real client would, rather than trying to inject raw packets that would need to separately replicate every side effect vanilla input handling already provides for free (sprint particle effects, click-cooldown timing, and so on).

## Solutions

Driving `KeyMapping`/click methods directly, rather than reimplementing Minecraft's own input-to-packet pipeline, means this file gets all of vanilla's existing correctness (timing, cooldowns, server-side validation expectations) without needing to duplicate any of it — a smaller, more robust surface than a from-scratch packet builder would be.

## Files Required

None beyond Minecraft/Forge's own client API and `DWClientMod` for logging.

## Files Used In

- [[WebSocketManager]] — `executeAction()`, the normal per-frame movement/control path.
- [[TCPServer]] (not yet read) — `executeInventoryAction()`/`executeScreenAction()`/`executeChatAction()`, per those three methods' own docstrings naming it as their caller.