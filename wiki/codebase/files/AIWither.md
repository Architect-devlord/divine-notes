---
type: file
status: ingested
---

# AIWither.java

💡 **Role**: the Wither god implementation — wither skull, a second "blue skull" variant, dash, summon wither skeletons, explosion, flight, transform/revert. Extends [[BaseGodEntity]]. **Confirmed, 2026-07-16, per [[GodSpawnHandler]]: never instantiated by the live spawn flow, which uses vanilla `EntityType.WITHER` instead.** See [[god-abilities]] for the full resolution.

**The same `"transform"`/`Float`-vs-expected-type mismatch already documented on [[AIEnderDragon]] appears here too, confirmed independently rather than assumed to generalize**: `case "transform"` matches a bare string the server never sends (`"transform_<mob>"` is the real convention), and would cast `params[0]` to `String` if it ever matched — a `ClassCastException` against the real runtime type (`Float`, from the wire protocol's `p1`/`p2`/`p3`). **A second, independent instance of the same cast-type problem, on a real, in-list ability this time — not just the transform special case**: `launchWitherSkull()` does `(boolean) params[0]` to decide whether this is a charged shot — also not a valid cast from `Float`, also a `ClassCastException` waiting to happen the moment [[GodEntityManager]]'s dispatch is ever fixed and `"wither_skull"` gets selected with a parameter present. All of this is currently moot regardless, given this class is never instantiated at all.

**A third, different kind of finding, worth being precise doesn't repeat the pattern above**: this file fully implements a sixth ability, `"blue_skull"` — its own cooldown field, its own `launchBlueSkull()` method, its own NBT persistence key — that **doesn't appear in [[action_format_sync]]'s `GOD_ABILITY_NAMES` table for Wither at all** (confirmed against that page's already-verified five-ability list: `wither_skull`, `dash`, `summon_wither_skeletons`, `explosion`, `fly`). Not a wire-format bug like the other two — a real, complete, well-written ability implementation the trained policy's action space has no index for, so it could never be selected even if every dispatch problem elsewhere were fixed.

## Fields & attributes

`skullCooldown`/`blueSkullCooldown`/`dashCooldown`/`summonCooldown`/`explosionCooldown`, `isFlying`/`isDashing`/`dashTicks`. `createAttributes()`: 300 HP (the highest of the six gods, not cross-verified against all others this pass), 20 attack damage, 4 armor.

## `useAbility(abilityName, params...)`

- `"wither_skull"` → `launchWitherSkull(params)` — the cast-mismatch case above; cone-based hit detection (`dot > 0.9` against look direction) within a 20-block box, 5 damage, or 8 + fire if the (currently unreachable) blue flag is true.
- `"blue_skull"` → `launchBlueSkull(params)` — the not-in-the-action-space ability above; tighter aim cone (`0.92`), 8 damage, fire, and a genuine `WITHER II` `MobEffectInstance` for 10s — a more complete implementation than the "blue" branch of `launchWitherSkull()` above it, which only adds fire, not the Wither effect itself. Worth noting as a small internal inconsistency independent of the reachability issue: two code paths for conceptually the same "charged skull" idea, with different actual effects.
- `"dash"` → `performDash()` — a 1-second (20-tick) forced-velocity charge, damaging anything within a 2×1×2 box around the entity each tick while dashing, not just on activation.
- `"summon_wither_skeletons"` → `summonWitherSkeletons()` — visual-only per its own in-line comment ("In production, spawn actual `WitherSkeleton` entities... for now, just visual effect") — three soul-particle columns arranged in a circle, no actual entities spawned.
- `"explosion"` → `createExplosion()` — radial damage/knockback falloff by distance within 8 blocks (up to 25 damage at the center), particle burst.
- `"fly"` → `toggleFlight(true)`.

## Problems (faced by traditional AI systems / LLMs)

Not AI/ML-specific — two distinct general problems, both already named on [[AIEnderDragon]]'s page and confirmed to recur here rather than being a one-off: a client/server parameter-encoding mismatch, and (new to this file) a Java-side ability implementation that was never registered in the Python-side action space that's supposed to enumerate every selectable ability.

## Solutions

Not solved in this file. The `summon_wither_skeletons` placeholder is at least honest about its own limitation, unlike the type-cast issues, which fail silently from the action space's perspective (the ability simply can never be chosen, no error visible anywhere) rather than announcing the gap.

## Files Required

- [[BaseGodEntity]].

## Files Used In

- [[GodEntityManager]] — confirmed dead dispatch, per that page.