---
type: file
status: ingested
---
# ServerGodAbilityExecutor.java

💡 **Role**: `execute(godPlayer, ability, level)` — the single entry point every god ability, for every god type, resolves through when triggered from the Python action-frame channel. **This is the file that fully resolves [[god-abilities]]'s open question**, read directly rather than inferred: `transform_<mob>`/`revert` are special-cased at the top of `execute()`, before the god-type dispatch switch, and hand off directly to [[GodDisguiseHandler]] — not a separate, competing path, but this executor's own first move for exactly those two ability shapes. Every other ability name falls through to one of six god-specific private methods (`executeWardenAbility`, `executeWitherAbility`, `executeEnderDragonAbility`, `executeElderGuardianAbility`, `executeOracleAbility`, `executeCreakingAbility`), each a `switch` over the ability names already catalogued on [[god-abilities]] and cross-checked there against [[action_format_sync]]'s `GOD_ABILITY_NAMES` table.

## Imports

A large Minecraft/Forge block (`ServerPlayer`, `ServerLevel`, `LivingEntity`, `AABB`, `Vec3`, particle/effect/damage types). `com.divineworld.DWMod`, `com.divineworld.events.GodDisguiseHandler`, `com.divineworld.commands.GodSpawnHandler` (referenced for the "body entity" associated with a god-controlling player — the visible boss-model puppet, not read this pass) — internal.

## The dispatch mechanism (the headline finding)

`execute(godPlayer, ability, level)`:

1. `if (ability.startsWith("transform_"))` — extracts the target mob name from after the prefix, calls `GodDisguiseHandler.applyTransform(godPlayer, targetMob, level)`, returns.
2. `if ("revert".equals(ability))` — calls `GodDisguiseHandler.removeTransform(godPlayer)`, returns.
3. Otherwise, dispatches to the god-specific method matching `godPlayer`'s current god type.

This method's own docstring states the reason for step 1/2 directly: _"Abilities include 'transform_<mob>' and 'revert' cases so Python AI can trigger transforms via the action frame channel."_ Confirms, precisely, that this class is the actual Java-side receiver of whatever main.py/god_controls.py's ability-index decoding produces — not a hypothesis, a stated fact in the source.

## Cooldown system

Plain NBT-backed, no separate data structure: `isOnCooldown(player, key)` / `setCooldown(player, key, ticks)` read/write `"cd_" + key` integers directly on the player's persistent data. `tickAbilityCooldowns(player)` (called once per player per tick, from wherever the main tick loop lives — not traced this pass) walks every `"cd_"`-prefixed key and decrements. Simple, and it means cooldowns persist naturally across a disconnect/relog for free, since they live in the same NBT that already persists the player entity.

## Shared utilities

`nearbyLiving(origin, radius)` — a bounding-box entity query excluding the origin itself. `ringParticles(level, center, particle, count, radius, yOffset)` — evenly-spaced particles around a circle, reused across several abilities for a consistent "ring burst" visual language.

## Ability execution pattern

Sampled via `executeWardenAbility`, representative of the other five: each `case` checks its own cooldown first (returning early if still active), performs the actual game-world effect, then sets its own cooldown. Effects range from straightforward (`darkness`: apply blindness+darkness `MobEffectInstance`s to everything within 20 blocks) to more involved (`sonic_boom`: a ray-marched cone out to 30 blocks, hurting and knocking back anything it crosses, with particles only on alternating steps to avoid overdrawing). **One ability pair worth calling out specifically for how it preserves agent agency**: `burrow`/`emerge` aren't a fixed-duration effect — `burrow` sets invisibility/no-physics/invulnerability and a `dw_burrowed` flag with no timer at all; the in-line comment is explicit that this is deliberate ("No auto-emerge — the AI agent sends 'emerge' when ready"), and `emerge` is a second, separate ability call that only does anything if `dw_burrowed` is currently true. The agent controls its own duration by choosing when to send the second ability, rather than the Java side imposing a fixed window.

## Problems (faced by traditional AI systems / LLMs)

Not AI/ML-specific — the interesting problem this file demonstrates is a general one for any system dispatching a wide, heterogeneous action space to different handlers: some actions need parameters beyond a fixed enum index (a transform needs to know _which_ mob), and encoding that parameter into the action name itself (`"transform_<mob>"`) rather than adding a separate parameter field is a real, working design choice with a real, traceable consequence — it can't fit into a fixed per-index lookup table the way every other ability does, which is exactly why it doesn't appear in [[action_format_sync]]'s table.

## Solutions

The string-prefix special case, checked first and returning immediately, keeps the parameterized-transform concern fully separate from the fixed-index dispatch below it — the six god-specific methods never need to know the transform system exists at all. NBT-backed cooldowns sidestep a whole category of "lost cooldown state on relog" bugs for free, as a side effect of where the data lives rather than anything designed specifically to solve that problem.

## Files Required

- [[GodDisguiseHandler]] — `applyTransform()`/`removeTransform()`, the transform/revert special case.
- `GodSpawnHandler.java` (not yet read) — `getGodEntity()`, used by at least the Warden's `burrow` case.

## Files Used In

- **Confirmed, 2026-07-16, and the news isn't good**: the designed-for caller — [[WebSocketManager]] and [[TCPServer]] dispatching a decoded action-frame ability trigger to [[GodEntityManager]]`.executeGodAbility()`, which would call `IGodEntity.useAbility()` — never actually reaches this class at all. [[GodEntityManager]] is confirmed dead by an exhaustive repo-wide search (its dispatch depends on a field that's never assigned anywhere in the codebase). So this method's own docstring claim — "so Python AI can trigger transforms via the action frame channel" — describes the intended design accurately, but the intended path is currently broken upstream of this class, not within it. This class itself has no bugs found this pass; it's correctly written for input it currently never receives from an AI agent's own decisions. See [[GodEntityManager]] for the full account, and [[god-abilities]] for how this fits into the broader picture.
- Not yet confirmed: whether a human-operator path exists that reaches `execute()` some other way, bypassing the broken client-side link entirely.