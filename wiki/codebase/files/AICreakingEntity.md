---
type: file
status: ingested
---

# AICreakingEntity.java

💡 **Role**: the one genuinely live, custom boss-body entity in the whole god system — spawned by [[GodSpawnHandler]] for the Creaking god specifically, and confirmed by its own doc comments to be exactly what [[ServerGodAbilityExecutor]]`.executeCreakingAbility()` operates on. `extends Monster` (not `Player` — a deliberately lightweight entity, unlike the `DWClientBot`-side `entity/gods/` package's `Player`-based approach), `implements GeoEntity` for GeckoLib animation.

## Imports

Minecraft/Forge `Monster`, `LivingEntity`, `Level`, `EntityDataAccessor`/`SynchedEntityData`, attribute types. GeckoLib's `GeoEntity`/animation types.

## Fields

`IS_UNDERGROUND`/`IS_ON_CEILING`/`TENTACLES_DEPLOYED` — three booleans defined as `SynchedEntityData` accessors, meaning they sync to every client automatically through Minecraft's own entity-data system, no custom packet needed at all — a much simpler answer to "how do I show state on other players' screens" than anything in the `DWClientBot`-side package, which had no equivalent mechanism.

## Constructor & goals

`createAttributes()`: 200 HP, 12 attack damage, matching [[GodSpawnHandler]]'s own spawn-time attribute boost exactly (the two independently agree). **No AI goal selectors registered at all** — the constructor's own comment states why directly: *"movement is driven by `GodControlHandler` syncing the puppet player's position to this entity every tick"* — this entity doesn't decide anything on its own; it's a visible shell whose position and animation state are driven entirely from outside.

## Animation — explicit, exhaustive mapping to real ability names

The class's own doc comments name every trigger's real-world source directly, not left implicit: `idle`/`walk`/`underground_move`/`ceiling_move` react to the synced boolean state; `attack` fires for `ServerGodAbilityExecutor`'s `"tentacle_whip"` (without tentacles deployed) specifically; a separate trigger covers the tentacles-deployed case. This is the most concrete, direct confirmation found this pass that [[ServerGodAbilityExecutor]]'s ability names are genuinely live and connected to a real, visible in-game effect for at least this one god — not just correctly *written*, but confirmed wired to something that actually renders.

## Methods

- `setUnderground(bool)` / `setOnCeiling(bool)` / `setTentaclesDeployed(bool)` — **each explicitly commented "called by `ServerGodAbilityExecutor`"** — the most direct, unambiguous confirmation in this whole investigation of which class actually calls into which.
- `registerControllers()` — the GeckoLib animation-controller registration, reacting to the synced booleans and to `triggerAnim()` calls from [[GodSpawnHandler]]`.triggerGodAnimation()`.
- `mobInteract()` — explicitly disabled (returns a pass/no-op result); this entity isn't meant to be interacted with by other players the way a normal mob would be.
- `defineSynchedData()` — registers the three boolean accessors.

## Problems (faced by traditional AI systems / LLMs)

Not AI/ML-specific — a clean answer to the general problem of showing a non-locally-controlled entity's internal state to every observing client: rather than a custom packet (compare `NetworkHandler.java`'s `MorphSyncPacket`, built for a completely different purpose — visual disguise sync), `SynchedEntityData` piggybacks on Minecraft's own existing entity-sync machinery, which every client already handles correctly for any entity.

## Solutions

No AI goals plus fully external position/animation driving is the right shape for "a visible body that's really being puppeteered by something else" — adding vanilla AI here would just be redundant work fighting against `GodControlHandler`'s own positioning every tick.

## Files Required

None beyond GeckoLib and Minecraft's own entity-data API.

## Files Used In

- [[GodSpawnHandler]] — the sole constructor, and `triggerGodAnimation()`'s only non-no-op case.
- [[ServerGodAbilityExecutor]] — the confirmed caller of every setter, per this file's own doc comments.