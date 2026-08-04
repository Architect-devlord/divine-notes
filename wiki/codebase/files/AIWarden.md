---
type: file
status: ingested
---

# AIWarden.java

💡 **Role**: the Warden god implementation — sonic boom, darkness, sniff (vibration detection), burrow/emerge, transform/revert. Extends [[BaseGodEntity]]. Its own header comment documents a real prior fix: it used to extend something in the `Monster` hierarchy, conflicting with `BaseGodEntity`'s `Player`-based inventory system; now cleanly `Player`-based throughout.

**Fully resolved, 2026-07-16, by [[GodSpawnHandler]]: this class is never instantiated by the live spawn flow, which uses vanilla `EntityType.WARDEN` instead.** This settles what was flagged here as an open discrepancy — not as a live conflict between two implementations, but as confirmation one side was always dead. `tick()` here has an unconditional, hardcoded auto-emerge: `if (burrowTicks >= 100) emergFromBurrow();` — 5 seconds, no way to extend it. This directly contradicted what's documented on [[ServerGodAbilityExecutor]] for its own, separate `burrow`/`emerge` implementation: no timer at all, deliberately, per that method's own comment — "the AI agent sends `emerge` when ready." [[GodSpawnHandler]]'s own header comment confirms the *live* mechanic (via [[ServerGodAbilityExecutor]], operating on the real vanilla `Warden` entity it spawns) has no auto-emerge for either Warden or Creaking. This class's conflicting timer was never a competing live behavior — it lived in a class nothing ever instantiates.

**The same `"transform"`/param-type mismatch already documented on [[AIEnderDragon]] and [[AIWither]] recurs here too — confirmed a third time.**

## Fields & attributes

`sonicBoomCooldown`/`darknessCooldown`/`sniffCooldown`/`burrowCooldown`, `isBurrowed`/`burrowTicks`/`isSniffing`, `lastVibrationPos`/`vibrationAge` (a simple "remember the last thing that moved or hit me" vibration-sense model, aged out after 100 ticks). `createAttributes()`: 500 HP (highest of the six), 30 attack damage, 10 armor + 4 armor toughness.

## `useAbility(abilityName, params...)`

- `"sonic_boom"` → `useSonicBoom(params)` — a stepped raycast (0.5-block increments, 30 blocks) checking entity proximity to each sample point rather than a true line intersection test; 25 damage, ignores armor (`damageSources().sonicBoom()`), strong knockback.
- `"darkness"` → `applyDarkness()` — blindness + darkness on everything within 20 blocks, 15s (300 ticks).
- `"sniff"` → `performSniff()` — scans for any nearby entity with nonzero velocity, updates `lastVibrationPos`, draws a particle line from Warden to target. Detection-only; doesn't act on what it finds beyond the visual/state update.
- `"burrow"` → `burrowUnderground()` — requires `onGround()`; sets invisible/invulnerable/no-physics, sinks slowly for the first second.
- `"emerge"` → `emergFromBurrow()` — restores physics, damages (20) and knocks back everything within 5 blocks, eruption particles.
- `"transform"`/`"revert"` — the recurring mismatch, per above.

## Other overrides

- `hurt(source, amount)` — burrowed state grants true invulnerability (returns `false`, damage doesn't apply at all, not just reduced); otherwise updates vibration tracking from the attacker's position before delegating to `super.hurt()`.
- `toggleFlight()` — explicitly a no-op; Warden doesn't fly, burrows instead.
- `getDimensions(pose)` — genuinely shrinks to normal player hitbox size while transformed and in humanoid form, full Warden size (0.9×2.9) otherwise.

## Problems / Solutions

Not AI/ML-specific — the same recurring client/server parameter-encoding problem named on [[AIEnderDragon]]'s page. Not solved in this file, and now confirmed moot regardless, since this class is never instantiated.

## Files Required

- [[BaseGodEntity]].

## Files Used In

- [[GodEntityManager]] — confirmed dead dispatch, per that page. Also never reached via [[GodSpawnHandler]]'s spawn flow, which uses the vanilla `Warden` entity type instead of this class.