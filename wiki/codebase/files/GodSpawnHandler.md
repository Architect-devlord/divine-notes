---
type: file
status: ingested
---

# GodSpawnHandler.java

💡 **Role**: spawns the real, visible boss entity for a god agent and tears it down on disconnect. **This is the file that fully and definitively resolves the architectural question raised across [[AIWarden]], [[AICreaking]], and [[GodEntityManager]]** — read directly, not inferred. A god is represented by **two** objects at once: the actual connected `ServerPlayer` (the bot's own account — made invisible, no-physics, flying: a puppeteer, not a puppet) and a separate boss-body entity spawned at the same position, tracked in a static `Map<UUID, Entity>` keyed by the player's UUID. **`spawnGodBody()` uses vanilla Minecraft entity types for five of the six gods** — `EntityType.WARDEN`, `WITHER`, `ENDER_DRAGON`, `ELDER_GUARDIAN`, and (for Oracle) `EVOKER` — **and a separate, `DivineWorld`-side custom class, `AICreakingEntity`, for the sixth.** It never constructs `BaseGodEntity`, `AIWarden`, `AIWither`, `AIEnderDragon`, `AIElderGuardian`, `AIOracle`, or `DWClientBot`'s own `AICreaking` — the whole `DWClientBot` `entity/gods/` package, confirmed extensively this pass, is never instantiated by the actual spawn flow at all. This is independent of, and in addition to, [[GodEntityManager]]'s already-confirmed dead client-side dispatch — two separate, sufficient reasons that whole package never runs live.

**This also fully resolves [[AIWarden]]'s flagged burrow/emerge discrepancy, cleanly** — not as a live conflict, but as confirmation one side was always dead. This file's own header comment states directly: *"Both Warden... and Creaking... share this same `dw_burrowed`-flag mechanic independently — each god agent controls when to emerge itself, no auto-emerge timer for either."* [[ServerGodAbilityExecutor]]'s real implementation (operating on the entities *this* file spawns) has no auto-timer for either god, matching what was already documented there. `AIWarden.java`'s conflicting 100-tick timer was never a competing live behavior — it lived in a class nothing ever instantiates.

## A self-corrected comment, worth citing as a clean example of the author's own care

The Creaking entity-type mapping comment is explicit about its own history: it used to say "→ WARDEN," a stale placeholder left over from before the custom `AI_CREAKING` entity existed — even though the actual `switch` body had already been updated to the real entity type for some time. Fixed for accuracy, with an explicit note that there was no behavior change, only a comment catching up to code that had already moved on. A second self-correction in the same file: the burrow-behavior comment used to describe only Warden's own-emerge-control behavior; updated once Creaking's own `"emerge"` case became genuinely selectable from Python's action space (cross-referencing `action_format_sync.py`'s `GOD_ABILITY_NAMES["creaking"]` by name) rather than being left incomplete.

## Fields

`GOD_ENTITY_MAP` — the one static registry this whole architecture depends on.

## Methods

- `onPlayerJoin(event)` — checks [[AgentConfigLoader]]`.getGodTypeForName()`; schedules `spawnGodBody()` 40 ticks later (letting the player's own login/world-load settle first).
- `spawnGodBody(player, godType, agentId)` — resolves the entity type, spawns it at the player's position, disables vanilla AI (a documented fix for `EnderDragon` specifically: `HOVERING` phase crashes with an NPE outside the End dimension via a null `getDragonFight()`, so it starts in `SITTING_SCANNING` instead), clears `WitherBoss`'s spawn-invulnerability so it's immediately hittable, tags the entity via `TaggedEntitySystem` (not yet read), registers with `DWNPCManager.registerGodPlayer()` (not yet read), stores the original god type in NBT (survives future transforms), clears any stale burrow flag from a prior session, then makes the real player invisible/no-physics/flying and boosts its own combat attributes to match the god type (the numbers here match what each `AI*.java` subclass's own `createAttributes()` declared, confirming those numbers were at least used as the reference even though the classes themselves aren't instantiated).
- `replaceGodBody(player, mobType)` — the transform mechanism: removes the old boss entity, spawns a new one of the requested type (falling back to a Forge-registry lookup for an arbitrary vanilla mob id if it's not one of the six named god types), re-tags it, and updates `dw_god_type` (leaving `dw_original_god_type` untouched, preserving what `revert` needs to restore). **Worth flagging, not yet resolved**: this operates on the same `GOD_ENTITY_MAP`/boss-entity concept [[GodDisguiseHandler]]`.applyTransform()` was already confirmed to be triggered for — whether these two methods are the same mechanism under two names, two cooperating halves of one system, or two genuinely separate transform implementations wasn't settled this pass and is worth checking directly.
- `getGodEntityType(godType)` / `resolveVanillaEntityType(mobId)` — the type-resolution table above, plus a generic Forge-registry fallback for transforming into an arbitrary vanilla mob.
- `getGodEntity(uuid)` — the accessor [[ServerGodAbilityExecutor]] and [[AICreakingEntity]]'s own doc comments confirm as their lookup point.
- `triggerGodAnimation(uuid, controller, animName)` — looks up the boss entity and, only if it's an `AICreakingEntity`, calls `triggerAnim()`; a no-op for every vanilla-entity god, with an explicit "Future: add more GeoEntity god types here" comment — Creaking is the only god with custom animation triggering currently wired up at all.
- `onPlayerLogout(event)` — clears burrow state defensively, removes the boss entity, cleans up the map entry.

## Problems (faced by traditional AI systems / LLMs)

Not AI/ML-specific — the general problem is representing one conceptual "character" as two separate game objects (an invisible controller and a visible body) without letting them drift out of sync, and doing so across a system (Minecraft/Forge) that wasn't originally designed around that split.

## Solutions

The `UUID`-keyed map is a simple, correct join between the two halves — anything holding a player's UUID can find their boss body. Explicit cleanup on both directions of the connection lifecycle (spawn on join, teardown on logout, including defensively clearing burrow state that might otherwise persist stale into a new session) avoids the two halves ever silently outliving each other.

## Files Required

- [[AgentConfigLoader]] — `getGodTypeForName()`.
- `TaggedEntitySystem.java`, `DWNPCManager.java` (both not yet read) — entity tagging and god-player registration.
- [[AICreakingEntity]] — the one custom entity type this file spawns.

## Files Used In

- [[ServerGodAbilityExecutor]] — `getGodEntity()`, confirmed via [[AICreakingEntity]]'s own doc comments naming that class as the caller of its setter methods.
- Presumably [[GodDisguiseHandler]] (relationship to `replaceGodBody()` not yet settled, see above) and a new class surfaced this pass, `GodControlHandler.java` — named directly in [[AICreakingEntity]]'s own comment as what syncs the invisible puppet player's position to the boss body every tick. Not yet read; the most promising remaining lead for how a client-decoded ability trigger might actually reach [[ServerGodAbilityExecutor]]`.execute()`, since [[GodEntityManager]] is confirmed not to be that path.