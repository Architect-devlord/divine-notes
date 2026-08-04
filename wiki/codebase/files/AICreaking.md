---
type: file
status: ingested
---

# AICreaking.java

💡 **Role**: the Creaking god implementation — underground/ceiling movement modes, tentacle deployment with a whip attack and low-health life-steal, transform/revert. Extends [[BaseGodEntity]], `implements GeoEntity` directly (the only one of the six with its own explicit GeckoLib wiring documented as a discrete, named fix, rather than inherited silently from the base class).

**Fully and definitively resolved, 2026-07-16, by [[GodSpawnHandler]]: this class is never instantiated by the live spawn flow at all.** This file's own header comment named a separate, server-side class, `DivineWorld`'s `AICreakingEntity`, as what its own GeckoLib rewrite was modeled after — and that turned out to be the real answer. [[GodSpawnHandler]]`.spawnGodBody()` spawns [[AICreakingEntity]] (a `DivineWorld`-side `Monster` subclass) for the live Creaking god, never this `DWClientBot`-side `Player`-subclass. [[AICreakingEntity]]'s own doc comments confirm its setter methods are called directly by [[ServerGodAbilityExecutor]] — the real, live, confirmed path. This class, and the whole `entity/gods/` package it belongs to, is dead for two independent, sufficient reasons: [[GodEntityManager]]'s confirmed-dead client dispatch, and now this — the entities themselves are never constructed by the spawn flow in the first place.

**The recurring `"transform"`/`Float`-cast mismatch appears here too — confirmed a sixth and final time, with zero exceptions across the whole package**, though now confirmed moot regardless. `useAbility()`'s `"transform"` case does `(String) params[0]`, identical in shape to every other god file checked. Worth noting the contrast within this same method: `"life_steal"` and `"tentacle_whip"` both correctly guard with `params[0] instanceof LivingEntity` rather than an unchecked cast — the same safe pattern already seen on [[AIEnderDragon]]'s `launchFireball()`.

## Fields

`isUnderground`/`isOnCeiling`/`isAngry`/`tentaclesDeployed`, `currentMode` (`MovementMode` enum: `NORMAL`/`UNDERGROUND`/`WALL_CLIMBING`/`CEILING` — `WALL_CLIMBING` declared but never actually set anywhere in this file), `tentacleController` (a private inner `TentacleController` class managing four `TentacleSegment`s arranged in a circle, purely for particle-effect positioning — its own in-line comment notes the actual visible tentacle animation now comes from GeckoLib's `.animation.json`, not this Java-side segment math). `createAttributes()`: 200 HP, 12 attack damage, full knockback resistance, 64-block follow range — matching [[AICreakingEntity]]'s own attributes exactly, confirming those numbers were at least used as the reference even though this class itself is never instantiated.

## `useAbility(abilityName, params...)`

- `"toggle_underground"` → requires `onGround()` to enter; `handleUndergroundMovement()` (called every `tick()` while underground) auto-exits the moment the block below stops being solid — a real, continuous safety check, not a one-time gate.
- `"toggle_ceiling"` → requires a solid block 3 above to enter; `handleCeilingMovement()` similarly auto-exits and otherwise applies a small constant upward force to keep the entity pressed against the ceiling.
- `"deploy_tentacles"`/(retraction happens automatically via `updateAngryState()`, not through a matching ability call) — `deployTentacles()`/`retractTentacles()` toggle `tentacleController`'s particle bursts.
- `"life_steal"` → heals self by draining a target below 20% HP (safe `instanceof` guard, per above).
- `"tentacle_whip"` → 8-block-range hit, 8 damage, knockback (safe `instanceof` guard).
- `"transform"`/`"revert"` — the recurring mismatch.

## Other overrides

- `tick()` — tentacle animation update, underground/ceiling movement handling, and `updateAngryState()`: auto-deploys tentacles once whoever last hurt this entity drops below 80% HP, auto-retracts once that condition clears — the one god-specific behavior in this whole package that reacts to combat state on every tick without an explicit ability call driving it.
- `toggleFlight()` — a no-op; Creaking moves via underground/ceiling modes instead.
- `getGodType()` → `"creaking"`. `getScale()` → `1.2f`.

## Problems / Solutions

Not AI/ML-specific — the same recurring client/server parameter-encoding problem named across this whole package, confirmed here for a sixth and final time with no exceptions. Not solved in this file — and now confirmed moot regardless, given [[GodSpawnHandler]]'s resolution.

## Files Required

- [[BaseGodEntity]].
- [[AICreakingEntity]] — the real, live entity this class's own header comment pointed to, and which [[GodSpawnHandler]] confirms is what actually gets spawned.

## Files Used In

- [[GodEntityManager]] — confirmed dead dispatch, per that page.

---

**This closes the full `entity/gods/` package** (`BaseGodEntity` + all six concrete gods). Every file confirmed the same `transform`/param-type mismatch; three of the six also confirmed the same mismatch on a real, in-list ability beyond just transform (`wither_skull`'s boolean cast, `teleport`'s double cast). [[GodSpawnHandler]] and [[AICreakingEntity]] together provided the definitive answer this whole package's own open question pointed toward: none of these seven files are ever instantiated by the live game. See [[god-abilities]] for the complete, final resolution.