---
type: file
status: ingested
---

# AIEnderDragon.java

💡 **Role**: the Ender Dragon god implementation — flight, dragon breath, fireball, perch, transform/revert. Extends [[BaseGodEntity]] for the shared player-mechanics/animation base; this file is purely the god-specific ability logic and attributes. **Confirmed, 2026-07-16, per [[GodSpawnHandler]]: never instantiated by the live spawn flow, which uses vanilla `EntityType.ENDER_DRAGON` instead.** See [[god-abilities]] for the full resolution.

**A second, independent bug found here, currently masked by [[GodEntityManager]]'s already-confirmed dead dispatch (and now also by [[GodSpawnHandler]]'s confirmation this class is never instantiated at all), but real and worth documenting precisely for whenever that gets fixed.** `useAbility()`'s `"transform"` case does `(String) params[0]` to read a target mob name as a separate parameter. Two things wrong with that, verified independently: (1) the switch matches a bare `"transform"` string, but [[ServerGodAbilityExecutor]]'s own confirmed wire convention is a combined `"transform_<mob>"` string — Java `switch` does exact string equality, so this case can never match what the server-side dispatch actually sends; (2) even setting that aside, `params[0]` would never actually be a `String` when called through the live wire path — [[GodEntityManager]]`.executeGodAbility()` is called from both [[WebSocketManager]] and [[TCPServer]] with three raw `float` values (`p1`/`p2`/`p3`, confirmed on both those pages), autoboxed to `Float` when passed into the `Object... params` signature. A `Float`-to-`String` cast isn't legal in Java — it would throw `ClassCastException`, not fail gracefully. Contrast `launchFireball()` below, which guards the same kind of access with `instanceof` rather than a direct cast, and degrades to a safe no-op instead.

## Imports

`com.divineworld.client.entity.ModEntities`, Minecraft entity/attribute/particle types.

## Fields & attributes

`isFlying` (default `true`), `isPerched`, `breathCooldown`/`fireballCooldown` (tick-based, decremented in `tick()`). `createAttributes()`: 200 HP, 15 attack damage, full knockback resistance, 128-block follow range — a genuinely tanky, aggressive baseline compared to the other five gods (not cross-compared against all of them this pass).

## `useAbility(abilityName, params...)`

- `"dragon_breath"` → `useDragonBreath()` — no params used, cooldown-gated, damages every `LivingEntity` within an 8×4×8 box in front of the dragon and sets them alight for 5s. Takes no target parameter at all — an area effect, not an aimed one.
- `"fireball"` → `launchFireball(params)` — cooldown-gated; **safely** checks `params.length > 0 && params[0] instanceof LivingEntity target` before using it, so a `Float` param (the real runtime type via the live wire path) just fails the `instanceof` check and the ability cooldown-resets without damaging anything, rather than throwing. In-line comment notes this is a placeholder ("In production, use `DragonFireball` entity") — direct-hit damage only, no actual projectile entity spawned.
- `"perch"` → `togglePerch()` — flips gravity/flying state, no params.
- `"fly"` → `toggleFlight(true)` — always enables, never disables, through this specific ability name (disabling flight only happens via `togglePerch()`).
- `"transform"`/`"revert"` — the mismatched case described above.

## Other overrides

- `tick()` — cooldown decrement, a slight constant upward force while flying and airborne, and a 10%-per-tick chance of a dragon-breath particle purely for ambient flight visual flavor (unrelated to the actual `dragon_breath` ability).
- `toggleFlight(enable)` — sets `isFlying`/no-gravity; disabling multiplies vertical velocity by `0.5` for a slow descent rather than an instant drop.
- `spawnTransformParticles()` — a portal-particle burst, an override of a [[BaseGodEntity]] hook (not read on that page directly, but clearly present and called from the shared transform machinery there).
- `getGodType()` → `"ender_dragon"`. `getScale()` → `4.0f`.
- `addAdditionalSaveData`/`readAdditionalSaveData` — persists the four ability-state fields via NBT.

## Problems (faced by traditional AI systems / LLMs)

Not AI/ML-specific — a plain client/server contract mismatch, the same general shape as several other findings across this whole ingest: two sides of a boundary (here, [[ServerGodAbilityExecutor]]'s wire convention and this file's own parameter expectations) were written to different, incompatible assumptions about how a parameterized ability should be encoded — one as a combined string, the other as a separate typed parameter.

## Solutions

Not solved in this file. `launchFireball()`'s `instanceof`-guarded access is the safer pattern already present *in this same file* for the same general problem (an ability that needs a runtime parameter of uncertain type) — the transform case could follow that same pattern instead of an unchecked cast, independent of and in addition to fixing the string-format mismatch itself.

## Files Required

- [[BaseGodEntity]] — the shared base this class extends.

## Files Used In

- [[GodEntityManager]] — would call `useAbility()`/`toggleFlight()` through the `IGodEntity` interface, if its dispatch weren't already confirmed dead.