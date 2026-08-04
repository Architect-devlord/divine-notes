---
type: file
status: ingested
---

# AIOracle.java

💡 **Role**: the (god-type) Oracle implementation — wisdom aura, foresight (danger prediction + entity sensing), teleport, healing wave, knowledge beam, flight, transform/revert. Extends [[BaseGodEntity]]. **Not** [[OracleSystem]]/[[LLMOracleBrain]]'s Wandering-Trader tutor (System B) — this is the RL-policy-driven god type (System A), per [[oracle-two-systems]]'s already-established split, confirmed consistent here: nothing in this file references Ollama, `LLMOracleBrain`, or anything from that other system. **Confirmed, 2026-07-16, per [[GodSpawnHandler]]: never instantiated by the live spawn flow, which uses vanilla `EntityType.EVOKER` instead** (a deliberate change away from a Wandering Trader, specifically to avoid visual confusion with the *other* Oracle system, which does use one).

**A third distinct variant of the recurring client/server parameter-type mismatch, confirmed here rather than assumed to be the same shape every time.** `teleportToLocation()`'s "specific coordinates" branch does `(double) params[0]` (and `[1]`, `[2]`). Java's unboxing-cast rules require the object's exact runtime wrapper type to match the target primitive's box (`Double` for `double`) — a `Float` (the real runtime type via the wire protocol) fails that cast with `ClassCastException`, the same failure mode as the `String`/`boolean` casts already found on [[AIEnderDragon]]/[[AIWither]], just against a third target type. `"transform"`'s own bare-string mismatch recurs here too. All moot regardless, now that this class is confirmed never instantiated.

## Fields & attributes

`wisdomAuraCooldown`/`foresightCooldown`/`teleportCooldown`/`healingCooldown`/`knowledgeBeamCooldown`, `isFlying` (default `true`, `setNoGravity(true)` in the constructor), `wisdomAuraActive`/`wisdomAuraTicks`, `predictedDangerPos`/`dangerPredictionAge`. `createAttributes()`: 150 HP (lowest of the six, not cross-verified against all others this pass), highest follow range (100 blocks) — a support/utility-leaning stat profile rather than a combat one.

## `useAbility(abilityName, params...)`

- `"wisdom_aura"` → `activateWisdomAura()` — 10s window (ticked in `tick()`), Regeneration + Resistance to every non-`Monster` entity within 15 blocks each tick, 30s cooldown. `isEnemy()` is a deliberately simple placeholder (`instanceof Monster`) with an in-line comment noting a real team/faction system would replace it in production.
- `"foresight"` → `useForesight()` — for every hostile entity within 50 blocks, linearly extrapolates 20 ticks of its current velocity to predict a future position, stores it, shows glow particles there. Detection/prediction only, no follow-up action taken on the prediction itself.
- `"teleport"` → `teleportToLocation(params)` — the mismatched coordinate path above; falls back to "20 blocks in the current look direction" if fewer than 3 params are present, which is also what happens in practice given the params-length check would pass (3 floats are always sent) but the cast would throw before ever reaching the fallback — worth noting the fallback branch is effectively unreachable via the live wire path for the same reason the primary branch fails, not a safety net for it.
- `"healing_wave"` → `useHealingWave()` — heals every non-enemy entity (including self) within 20 blocks by 10 HP, 10s cooldown.
- `"knowledge_beam"` → `useKnowledgeBeam(params)` — accepts params but never reads them; fires in the current look direction only. 40-block stepped beam, 12 armor-ignoring magic damage plus a slow effect, 4s cooldown.
- `"fly"` → `toggleFlight(true)`.
- `"transform"`/`"revert"` — the recurring mismatch.

## Other overrides

- `toggleFlight(enable)` — sets both `isFlying` and `setNoGravity`.
- `getScale()` → `1.0f` — human-sized, unlike every other god in this package.

## Problems / Solutions

Not AI/ML-specific — the same general client/server parameter-encoding problem named on [[AIEnderDragon]]'s page, now confirmed in a third distinct form. Not solved in this file.

## Files Required

- [[BaseGodEntity]].

## Files Used In

- [[GodEntityManager]] — confirmed dead dispatch, per that page. Also never reached via [[GodSpawnHandler]]'s spawn flow.