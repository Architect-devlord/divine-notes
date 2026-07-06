---
type: system
status: ingested
---

# Event Handlers

💡 **What this is**: the three Forge event-bus listeners that aren't already covered elsewhere — `DWEventHandler` (server: join/leave/tick), `ClientEventHandler` (client: world-join/respawn/logout), `ClientChatEventHandler` (client: chat forwarding). `BreedingEventHandler`, `GodControlHandler`, `GodSpawnHandler`, and `GodDisguiseHandler`'s own `@SubscribeEvent` methods are covered on their respective pages.

## `DWEventHandler` — server-side join/leave/tick

`agents.json` is the sole authority on what a connecting username *is* — `TaggedEntitySystem.detectAgentType()` reads it once per fresh login and caches the result in NBT so every later tick-level check is an O(1) NBT read rather than a re-lookup. Two fixed bugs recorded in the class's own header are worth knowing:

- **God registration used to happen twice.** `onPlayerJoin` still calls `detectAgentType()` for a GOD (so `dw_god_type` is cached in NBT before `GodSpawnHandler.onPlayerJoin` reads it the same tick), but it no longer calls `registerGodPlayer` itself — [[wiki/codebase/god-abilities|GodSpawnHandler]] is now the sole owner of that, called only after the boss body actually exists in the world.
- **Ability cooldowns never decremented.** `ServerGodAbilityExecutor` stores per-ability cooldowns as NBT ints (`cd_sonic_boom`, etc.), but nothing was ticking them down — every ability became permanently unusable after its first use. `onServerTick` now calls `ServerGodAbilityExecutor.tickAbilityCooldowns(player)` once per god player per tick; the method just decrements every `cd_*` key by 1, floored at 0. A second `onServerTick` fix in the same pass wires up `GodDisguiseHandler.tickMorphBodies()`, which had the identical problem — a real-player (op-4) morph body would spawn once and then never follow its owner again, since nothing was calling the method that syncs its position each tick.

`notifyBackend()` (fire-and-forget, own daemon thread) is also where a "connected" event's response gets read back for a `spawn_pos` field — if present, it schedules a teleport back onto the main server thread (`srv.execute()`, since `teleportTo` isn't thread-safe) so an agent spawned via the API actually lands at the coordinates the spawn call asked for, instead of vanilla world spawn or its last bed.

## `ClientEventHandler` — client-side world-join/respawn/logout

Port resolution is the core job here: once `Minecraft.getInstance().getUser().getName()` is available (reliably true right after world-join), it's looked up against `agents.json` (falling back to the entity name if the session name doesn't match) to get a TCP port, with the WebSocket port always `TCP + WS_PORT_OFFSET`. `TCPServer` and `WebSocketManager` are only started *after* that resolution completes, specifically so they bind the right port from the start rather than a placeholder. If the agent is a god type, `GodEntityManager.initializeGodEntity()` fires here too.

A fixed NPE (`Bug G`) is worth knowing about if this code gets touched again: `onPlayerRespawn` used to call `initializeGodEntity()` without checking that `mc.level` was non-null yet, which could crash on respawn. Now it guards both `mc.player`/`mc.level`, and if `level` isn't ready yet, defers by resetting `initialized = false` so the next client tick retries the whole world-join sequence rather than the respawn path specifically.

## `ClientChatEventHandler` — proximity chat into the agent's ears

Short and single-purpose: every `ClientChatReceivedEvent` is checked against the vanilla proximity-chat format (`"<Speaker> message"` — anything not starting with `<` is a system message and ignored), parsed into speaker/body, and forwarded via `WebSocketManager.sendChatObservation()` — unless the speaker is the agent itself (echo suppression by exact name match). Its own header comment traces the full round trip: [[wiki/codebase/architecture-overview|ProximityChatHandler]] on the server broadcasts a vanilla system message to nearby players → this class catches it client-side → the WebSocket carries it to `/ws/agent` as a JSON text frame the cognitive loop can hear.
