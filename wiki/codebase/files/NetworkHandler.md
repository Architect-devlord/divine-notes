---
type: file
status: ingested
---
# NetworkHandler.java

💡 **Role**: the DivineWorld server mod's Forge networking channel — and, read directly, narrower than its name suggests. It registers and sends exactly **one** packet type, `MorphSyncPacket` (server → client), purely to sync a god's disguised appearance to nearby players' clients _after_ a transform has already been decided and applied server-side. **Not** the dispatch point for ability triggers arriving from Python — that question is fully resolved on [[god-abilities]], and the answer doesn't run through this file at all. This class's own header comment is a useful, explicit negative confirmation: _"Chat bubble system removed — proximity chat handles all agent speech."_

## Imports

`net.minecraft.resources.ResourceLocation`, `net.minecraft.server.level` (`ServerLevel`, `ServerPlayer`), `net.minecraftforge.network` (`NetworkRegistry`, `PacketDistributor`, `simple.SimpleChannel`) — Minecraft/Forge. `java.util.UUID`, `java.util.function.Predicate` — stdlib. `com.divineworld.DWMod` — internal.

## Fields

`PROTOCOL_VERSION` (`"2"`), `INSTANCE` (the one `SimpleChannel`), `registered` (guards against double-registration, which the in-line comment notes causes crashes), `packetId` (an auto-incrementing counter behind the private `id()` helper — currently only ever called once, for packet ID 0).

## Methods

- `register()` — builds the channel with a version-check predicate. **A real, documented fix worth preserving precisely**: the predicate used to require an _exact_ match on `PROTOCOL_VERSION` on both sides, with no allowance for a client that has no channel at all — `NetworkRegistry.ABSENT` is Forge's own documented sentinel for that case, and accepting it is specifically what lets a vanilla client (or anyone without this mod) complete the login handshake instead of being rejected outright. The fix comment is explicit that `ChannelBuilder`/`.optional()` (a Forge 1.20.4+ API) isn't needed for this version — `NetworkRegistry.newSimpleChannel()` is already correct and current for 1.20.1; only the predicate itself needed to change.
- `broadcastMorph(transformedPlayer, level, newMobType)` — the one thing this channel actually does: builds a `MorphSyncPacket` and sends it to every player within 64 blocks. **Another specific, documented fix**: rather than assuming `SimpleChannel.send()` silently no-ops for a player whose client lacks the channel (nothing in Forge's docs guarantees that), it checks `INSTANCE.isRemotePresent(nearby.connection.connection)` first — a real, if narrow, guard, since the version-predicate fix above means almost any player who completes login already has the channel.

## Problems (faced by traditional AI systems / LLMs)

Not AI/ML-specific — both fixes in this file are general networking-protocol-compatibility problems: a version-negotiation predicate that's stricter than it needs to be locks out clients that should be allowed to connect, and assuming an unverified API guarantee (silent no-op on a missing channel) is exactly the kind of assumption that's cheap to just check instead.

## Solutions

Accepting Forge's own documented `ABSENT` sentinel, rather than requiring an exact version string, is the correct fix for compatibility without weakening the actual version check for clients that _do_ have the mod. Checking `isRemotePresent()` before sending, rather than trusting an assumed no-op, is a cheap, correct guard regardless of how often it actually matters in practice.

## Files Required

- `MorphSyncPacket.java` (not yet read) — the packet class itself, encode/decode/handle.
- `com.divineworld.DWMod` — logger access.

## Files Used In

- [[GodDisguiseHandler]] — the sole caller of `broadcastMorph()`, after every successful `applyTransform()`/`removeTransform()`.