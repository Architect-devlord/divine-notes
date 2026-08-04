---
type: file
status: ingested
---

# BaseGodEntity.java

💡 **Role**: the shared base every one of the six god entity types extends — and a genuinely significant architectural fact worth leading with: `BaseGodEntity extends Player`, not `Mob`. A god entity *is* a (fake, offline-profile) player as far as Minecraft's own systems are concerned, which is what makes full inventory, mining, crafting, trading, and enchanting available essentially for free by inheritance, rather than needing to be reimplemented from scratch. `implements IGodEntity, GeoEntity` layers the mod's own ability contract and GeckoLib's animation system on top.

**Confirmed, 2026-07-16, by [[GodSpawnHandler]]: this whole class and its six subclasses are never instantiated by the live spawn flow at all.** `GodSpawnHandler.spawnGodBody()` uses vanilla Minecraft entity types (`Warden`, `Wither`, `EnderDragon`, `ElderGuardian`, `Evoker`) for five gods and a separate `DivineWorld`-side class, `AICreakingEntity`, for the sixth — never this `DWClientBot`-side `Player`-based hierarchy. This is independent of, and in addition to, [[GodEntityManager]]'s already-confirmed dead client-side dispatch. See [[god-abilities]] for the full, definitive resolution.

## Imports

A large Minecraft/Forge block (entity, item, block, level, sound, enchantment, trading, projectile types) plus `software.bernie.geckolib.*` (GeckoLib's animation framework) and `com.mojang.authlib.GameProfile`. `com.divineworld.client.entity.IGodEntity` — internal. See [[IGodEntity]].

## Fields

`geoCache` — one `AnimatableInstanceCache` per entity instance, required by GeckoLib. `godInventory` — a real `Inventory` (Player's own class, not a generic container — the in-line comment notes this specifically matters because `Inventory` has `getDestroySpeed()`, which a simpler container wouldn't). `isTransformed`/`transformedMobName` — the mob-disguise state this class persists (see [[GodDisguiseHandler]] for where it's set from the server side). `canMineBlocks`/`canPlaceBlocks`/`canCraft`/`canUseItems` — per-instance capability flags. `useItem`/`useItemRemaining`/`attackStrengthTicker` — Player-mechanic bookkeeping.

## Constructor & lifecycle

- `BaseGodEntity(type, level)` — constructs with a random-UUID `GameProfile` named `"GodEntity"` and `BlockPos.ZERO` (the real spawn position comes from wherever the server-side spawn logic places it, not this constructor).
- `tick()` — calls `super.tick()`, ticks item-use countdown and inventory. **A documented, named fix (`M-06`) worth preserving precisely**: this method used to also increment `attackStrengthTicker` itself, on top of `Player.tick()` already doing so internally — doubling the attack-cooldown recharge rate, so every *second* swing landed as a premature critical hit rather than every swing after the intended full cooldown. Fixed by simply removing the duplicate increment and trusting the inherited one.

## Player-mechanic reimplementation (mining, placement, item use, interaction, crafting, trading, enchanting)

All in-line commented `"PRODUCTION READY"` — a full, careful reimplementation of the mechanics `Player`'s own subclasses normally get from vanilla input handling, since a god entity has no real client input driving it. Mining (`mineBlock`) reads tool correctness and applies the same efficiency/water/airborne speed penalties `getDestroySpeed()` vanilla logic would. Placement (`placeBlock`) builds a real `UseOnContext`/`BlockPlaceContext` pipeline rather than a shortcut, so blocks with orientation (stairs, logs, pistons) place correctly. Item use (`useItem`) branches on food/potion/projectile item types. `interactOn`/`interactWithBlock` mirror vanilla's own two-phase "let the target/block handle it first, then try the held item" interaction order. `craftItem`/`trade`/`enchantItem` are complete, independent implementations of crafting-table-free crafting, villager trading (one trade per call, with XP reward), and XP-gated enchanting — none delegated to vanilla UI, since a god entity has no player screen to open one from.

## Persistence

`addAdditionalSaveData`/`readAdditionalSaveData` — transform state, attack ticker, and the in-progress use-item all round-trip through NBT alongside the inventory (via `Inventory.save()`/`.load()`, Player's own methods).

## GeckoLib animation (humanoid form)

A documented, two-controller design, explained precisely in this class's own header comment: `base_controller` (5-tick blend) is a looping movement state machine — swim/sneak/run/walk/idle, read from real physical state (`isUnderWater()`, `isCrouching()`, horizontal speed threshold `0.22`) — active only when the humanoid puppet render mode is on (`dw_form = "humanoid"`, see [[GodDisguiseHandler]]'s form-cycle system); the boss-body form uses `GodEntityRenderer`'s existing player-model path instead and doesn't need this. `ability_controller` (0-tick, instant) is one-shot triggered animations — `attack`/`hit`/`mount` registered here, with `registerExtraAbilityTriggers()` as an explicit extension point subclasses override to register their own god-specific triggers without needing to override the whole controller-registration method. The header is explicit that every animation name here has to match a keyframe name in that god's own `god_*.animation.json` file exactly, since GeckoLib drives the humanoid puppet's geometry from scratch (unlike the boss-body form, which reuses vanilla's player model and doesn't need matching keyframes for the basics).

## Abstract contract for subclasses

`getGodType()`, `useAbility(String, Object...)`, `toggleFlight(boolean)` — the three methods [[IGodEntity]] requires and this class leaves unimplemented, one per concrete god. **Directly relevant to the confirmed-dead client dispatch documented on [[GodEntityManager]]**: `useAbility()` is where each god's actual ability logic lives, and it's currently unreachable through the live action-frame channel regardless of what any individual implementation does, since nothing ever populates the `currentGodEntity` reference needed to call it.

## Problems (faced by traditional AI systems / LLMs)

Not AI/ML-specific — the general problem is giving a non-human-controlled entity the same mechanical capabilities a real player has (inventory, mining, crafting) without a real client to drive vanilla's own input-handling code, which all assumes a `LocalPlayer`/`ServerPlayer` pair with a real network connection behind it.

## Solutions

Extending `Player` directly, rather than building a `Mob` with hand-rolled equivalents of inventory/mining/crafting, means this class gets the *correct* underlying data model (a real `Inventory`, real `ItemStack` semantics) for free — the reimplementation work here is specifically the parts that normally come from client input (deciding *when* to mine, place, or trade), not the underlying mechanics themselves, which stayed inherited. The GeckoLib two-controller split (looping state machine vs. one-shot triggers) cleanly separates "what am I doing right now" from "what did I just do once," which map naturally onto continuous movement state and discrete ability activations respectively.

## Files Required

- [[IGodEntity]] — the interface contract this class implements the shared parts of and leaves the rest abstract.

## Files Used In

- The six concrete subclasses: `AIWither`, `AIEnderDragon`, `AIWarden`, `AIElderGuardian`, `AIOracle`, `AICreaking` (all confirmed dead in the live spawn flow, per [[GodSpawnHandler]]).
- [[GodEntityManager]] — would call `useAbility()`/`toggleFlight()` through the `IGodEntity` interface, if it ever held a live reference (confirmed it doesn't).