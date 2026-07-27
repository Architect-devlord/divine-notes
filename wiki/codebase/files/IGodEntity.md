---
type: file
status: ingested
---
# IGodEntity.java

💡 **Role**: the interface contract every god entity type implements client-side — 41 lines, an interface only, no logic of its own. `useAbility(String, Object...)` is the method [[GodEntityManager]]`.executeGodAbility()` would call, if it ever reached one — confirmed on that page that it never does, since [[GodEntityManager]] never holds a reference to any implementer of this interface.

## Methods

- `useAbility(abilityName, params...)` — the per-god ability implementation contract. No default; every implementer must provide its own.
- `toggleFlight(enable)` — no default either.
- `addPlayerInventory()` — grants player-style inventory capability to a god entity; no default.
- `isInPlayerForm()` — defaults `false`.
- `getGodType()` — defaults `"unknown"`.

## Problems / Solutions

Not applicable — a pure interface, no logic to analyze.

## Files Required

None.

## Files Used In

- `com.divineworld.client.entity.gods.*` (`AIWither`, `AIEnderDragon`, `AIWarden`, `AIElderGuardian`, `AIOracle`, `AICreaking`) — the six implementers, none read this pass.
- [[GodEntityManager]] — the intended, currently-unreachable caller of `useAbility()`.