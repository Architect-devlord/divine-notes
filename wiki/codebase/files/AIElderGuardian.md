---
type: file
status: ingested
---

# AIElderGuardian.java

💡 **Role**: the Elder Guardian god implementation — mining fatigue, a charge-then-fire laser beam, thorn-style damage reflection, a defensive spikes mode, permanent water breathing, transform/revert. Extends [[BaseGodEntity]]. **Confirmed, 2026-07-16, per [[GodSpawnHandler]]: never instantiated by the live spawn flow, which uses vanilla `EntityType.ELDER_GUARDIAN` instead.** See [[god-abilities]] for the full resolution.

**The same `"transform"`/param-type mismatch already documented on [[AIEnderDragon]], [[AIWither]], and [[AIWarden]] recurs here too — confirmed a fourth time, not assumed to generalize from three.** Given the consistency across every file checked so far, this reads as a systemic issue in how the ability contract was implemented across the whole package, not a one-off slip in any single file — moot regardless, now that the class is confirmed never instantiated.

**One small, low-stakes timing bug found here specifically**: `hurt()`'s thorn-reflection check is `thornCooldown < 100 && thornCooldown > 0`. `activateThornAttack()` sets `thornCooldown = 100` on activation, then `tick()` decrements it. On the very tick reflection is activated, `thornCooldown` is still exactly `100` — which fails the `< 100` half of the check, so the first tick of the ability's own 5-second window doesn't actually reflect anything. A one-tick gap in a 100-tick window; likely imperceptible in practice, but a real, verifiable off-by-one.

## Fields & attributes

`miningFatigueCooldown`/`laserBeamCooldown`/`thornCooldown`/`spikesCooldown`, `spikesActive`/`spikesTicks`, `isLaserCharging`/`laserChargeTicks`. Constructor grants permanent `WATER_BREATHING` (`Integer.MAX_VALUE` duration) directly, not gated behind any ability. `createAttributes()`: 250 HP, 18 attack damage, 8 armor.

## `useAbility(abilityName, params...)`

- `"mining_fatigue"` → `applyMiningFatigue()` — `DIG_SLOWDOWN` level III (amplifier `2`) for 5 minutes (6000 ticks), everything within 30 blocks, 60s cooldown.
- `"laser_beam"` → `chargeLaserBeam()` — only sets a charging flag; the actual `fireLaser()` call happens from `tick()` after a fixed 2-second (40-tick) charge completes, with charging particles drawn along the look vector each tick in the meantime. The ability call and the ability's actual effect are two ticks apart by design, not a bug — the charge-up is the point.
- `"thorn_attack"` → `activateThornAttack()` — the off-by-one case above; doesn't do anything itself beyond starting the cooldown, since the actual reflection logic lives entirely in `hurt()`.
- `"guardian_spikes"` → `activateGuardianSpikes()` — 10-second (200-tick) window, ticked in `tick()`: continuously deals 2 damage to anything within 3 blocks each tick (not just on activation), 20s cooldown.
- `"transform"`/`"revert"` — the recurring mismatch, per above.

## Other overrides

- `hurt(source, amount)` — the thorn-reflection check (with its off-by-one), reflecting half the incoming damage back at a `LivingEntity` attacker before delegating to `super.hurt()`.
- `toggleFlight()` — a no-op; in-line comment notes water mobility could be a future direction here instead of flight.

## Problems / Solutions

Not AI/ML-specific — same general client/server parameter-encoding problem already named on [[AIEnderDragon]]'s page, confirmed recurring rather than isolated. Not solved in this file.

## Files Required

- [[BaseGodEntity]].

## Files Used In

- [[GodEntityManager]] — confirmed dead dispatch, per that page. Also never reached via [[GodSpawnHandler]]'s spawn flow.