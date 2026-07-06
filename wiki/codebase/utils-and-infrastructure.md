---
type: system
status: ingested
---

# Utils & Agent Registry

💡 **What this is**: the small infrastructure classes underneath everything else — agent tagging/detection, the genesis/divine-reset ritual, the books that explain it in-game, and the particle-circle effect both rituals share. `AStarPathfinder`, `AgentConfigLoader`, and `Config` are already covered elsewhere; `Logger` (a four-method try/catch wrapper around `DWMod.LOGGER` so other classes can log without a direct dependency) and `DebugFlags` (a single `Set<UUID>` toggling a per-player actionbar debug overlay) are exactly as small as they sound and don't need more than this sentence each.

## `TaggedEntitySystem` — the single source of truth for "what is this player"

Everything else in the mod (`DWNPCManager`, `DWEventHandler`, every command) ultimately asks this class, directly or via NBT it already wrote. `detectAgentType()` is the one entry point: NBT cache first (an O(1) read if the player has joined before), then `agents.json` on a fresh join, and username-not-found falls through to `REAL_PLAYER`. A fixed bug worth knowing (`B-03`): god players need `dw_gender = "dual"` in NBT — not male/female — so [[breeding-system]]'s proximity scan includes them and Python's role-resolution can treat either partner as either role; pre-fix saves that predate this could be missing the key, so `detectAgentType()` re-checks and repairs it on every login rather than assuming a god's NBT is already correct.

Below that, a generic tag registry (`tagEntity`/`hasTag`, backed by both per-entity NBT and an in-memory `TAG_INDEX` for fast "all entities with tag X" world queries) that `dw_npc`/`dw_god`/`dw_genesis_immune` all ride on top of, plus the god-specific fields (`godType`, `divinePower`).

## `DWNPCManager` — the registry callers actually use

Thin, mostly-delegating wrapper over `TaggedEntitySystem` (`registerAIPlayer`/`registerGodPlayer`/`isAIPlayer`/`isGodPlayer`/`promoteToGod`/`demoteFromGod`) plus one thing that's genuinely its own: `sendAgentChat()`, a per-agent 1-second chat rate limit (`tickCooldowns()`, ticked from [[events|DWEventHandler]]'s server tick) that only delivers to players within `ProximityChatHandler.PROXIMITY_RADIUS` blocks. Its own header notes a fixed bug here too (`S-04`): this used to broadcast to the entire server rather than scoping to proximity, which didn't match how players could actually *hear* an agent speak in-world.

## `GenesisManager` — the ritual, the cooldown, and the crash it used to cause

Three ways to trigger genesis, all converging on the same `PythonBackendClient.spawnGenesisAgents()` call: the `/genesis` command (in [[commands]]), right-clicking a held Genesis Codex book, or right-clicking one lying on the ground. That third path has its own fixed-crash story worth keeping around: `PlayerInteractEvent.EntityInteract` fires on *both* logical sides, and the original handler ran server-only calls (`PythonBackendClient`, `ServerLevel` methods) without checking which side it was on — a guaranteed `NullPointerException` on the client. The fix is a one-line `isClientSide()` guard at the top; everything below it is now server-only, and `setCanceled(true)` stops vanilla item pickup so the book stays on the ground after the ritual fires (matching the block-click path's `event.setCanceled(true)` for the same reason).

Divine Reset is a scripted 200-tick (10-second) countdown (`onServerTick`), broadcasting warnings at specific tick marks (40/100/160 → "8/5/2 seconds..."), spawning a [[#DivineMagicCircle — one particle-ring renderer, two rituals|magic circle]] frame every `CIRCLE_INTERVAL` (3) ticks, then at tick 200 disconnecting every AI/god player with an in-character message and notifying Python so their memories actually get purged — the countdown itself is purely cosmetic tension; nothing is destructive until the final tick.

## `BookFactory` — in-world documentation, not functional items

Three `WRITTEN_BOOK` items (`genesisCodex()`, `firstFlameBook()`, `commandReferenceCard()`), each just NBT pages of formatted `Component` text — no click handlers or custom behavior live here. Worth noting the class's own comment: these used to *do* something (presumably trigger genesis directly), and are now explicitly "just informational items" — reading `genesisCodex()` tells the player to go run `/genesis` themselves rather than the book performing the action.

## `DivineMagicCircle` — one particle-ring renderer, two rituals

Stateless: `spawnGenesisCircle(level, center, tick)` and `spawnDivineResetCircle(level, center, tick)` each draw one animated frame of a layered, 48-point, 5-block-radius particle ring — the caller (`GenesisManager`) is responsible for scheduling repeated calls across ticks to get a continuous animation, this class has no timer of its own. The two rituals are visually distinct by design: Genesis uses light/creation particles (`END_ROD`, `PORTAL`, `ENCHANT`, `DRAGON_BREATH`, `TOTEM_OF_UNDYING`); Divine Reset uses fire/decay particles (`SOUL_FIRE_FLAME`, `FLAME`, `SMOKE`, `ASH`, `SCULK_SOUL`) — the palette itself signals which ritual is running before a player reads any chat message.
