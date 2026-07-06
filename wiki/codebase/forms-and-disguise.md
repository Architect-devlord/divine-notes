---
type: system
status: ingested
---
# Forms & Disguise

💡 **What this is**: how a god's `transform`/`revert` ability (every god has both — see [[god-abilities]]) actually manifests visually, plus a separate, broader disguise system it's easy to conflate with it. Pulled from the same commit that gave all six gods full Blockbench 3D models — the first time this wiki has looked at the rendering layer at all.

## Two forms, one god

Every God Agent has a **humanoid form** and a **true form**, and the `transform`/`revert` abilities every god's table includes (see [[god-abilities]]) toggle between them.

- **Humanoid form** renders through a single shared class, `GodHumanoidGeoModel<T extends BaseGodEntity>` — it doesn't hardcode per-god assets, it derives the resource path from `entity.getGodType()`: `geo/entity/god_{type}.geo.json`, `textures/entity/god_{type}.png`, `animations/entity/god_{type}.animation.json`. One class, six gods, by construction.
- **True form** uses each god's own dedicated boss-model assets — e.g. Creaking's true form uses `ai_creaking.*` via its own `CreakingGeoModel`/`CreakingGeoRenderer`, separately registered, not routed through the shared humanoid class at all.
- **Animation names are shared across both forms on purpose.** The same triggerable names (`attack`, `burrow`, `tentacles_out`, etc.) exist in both the humanoid and true-form animation files, specifically so `BaseGodEntity.registerControllers()`'s animation-triggering code doesn't need to know or care which form is currently active — it fires the same name either way, and whichever form's `.animation.json` is currently loaded provides the actual keyframes.

## GodDisguiseHandler — a broader, separate system

Easy to conflate with the above, but distinct: `GodDisguiseHandler.java` (server-side) lets a god disguise as an **arbitrary vanilla mob**, not just switch between its own two forms. Gated to gods and level-4 operators only (`canTransform()`). A `FORBIDDEN_TYPES` set excludes anything that wouldn't make sense as a disguise body — `area_effect_cloud`, `falling_block`, `item`, `item_frame`, `end_crystal`, `armor_stand`, `experience_orb`.

**A fixed bug worth knowing about**: `replaceGodBody()` writes the transform target into `dw_god_type` (e.g. `"warden"`), which meant `removeTransform()` — reading from that same key to figure out what to revert _to_ — would restore the wrong body (a warden, when the god was actually an oracle that had disguised as a warden). Fixed by having `GodSpawnHandler.spawnGodBody()` additionally write a separate `dw_original_god_type` key that `replaceGodBody()` never touches, and having `removeTransform()` read exclusively from that instead. The general lesson: when one system's "current state" key doubles as another system's "state to restore" key, a write from the first silently corrupts the second's assumptions.

A second fixed bug: morph bodies (a real player disguised as a mob) were being set invulnerable, which doesn't match how morphing is supposed to behave for a real player — removed.

## TransformationHandler — the client-side visual layer

Purely cosmetic (particle bursts) and explicitly has **zero imports from the server mod** — the actual entity swap is a real server-side entity replacement (`GodSpawnHandler.replaceGodBody()`), which every client already sees for free because it's a genuine world entity change, not a client-side illusion. This class only adds particles on top of a change that already happened.

Tracks a **3-way form cycle** distinct from ordinary mob-morphing: packets prefixed `"form:"` (`form:god` | `form:humanoid` | `form:disguise`) are routed to a separate particle/state path (`FORM_STATE`/`FORM_PARTICLE_TICKS`) from ordinary morph packets (`MORPH_STATE`/`PARTICLE_TICKS`), specifically so a form-change is never misread as "the god morphed into a vanilla mob literally named 'god' or 'humanoid'" by the existing `isMorphed()`/`getMorphType()` logic.

**Confirmed, and corrected from an earlier guess**: the `form:disguise` broadcast is not `GodDisguiseHandler.applyTransform()`'s doing at all — it comes from a different method entirely, `applyGodForm()` (called by `cycleGodForm()`, the `god → humanoid → disguise → god` cycle, itself a system the class explicitly separates from mob-morphing via its own `dw_form` NBT key, distinct from `dw_disguised`/`dw_disguise_type`). `applyGodForm()` calls `NetworkHandler.broadcastMorph(player, level, "form:" + form)` directly — that literal string concatenation is the entire wiring. `applyTransform()`, by contrast, broadcasts the raw mob type with no prefix (`broadcastMorph(player, level, normType)`, e.g. `"warden"`) and lands in `TransformationHandler`'s other map (`MORPH_STATE`) via the plain `else` branch, with its own god-type-specific particle set. Both paths share the same transport (`NetworkHandler.broadcastMorph` → `MorphStateCache` → drained on `ClientTickEvent`) and the same client-side class, but are otherwise two independent systems that happen to look like one from the outside.

## The assets themselves

Now covered separately: [[blockbench-assets]] — the actual `.bbmodel`/`.geo.json`/`.animation.json` files for all six gods plus the Civilian Agent's own humanoid model, and confirmation that the humanoid form's uniform bone rig and each true form's dedicated, much larger rig match this page's code-level description exactly.

## How the three-way form actually gets on screen

`GodEntityRenderer` is the one renderer Forge actually has registered for the player-puppet entity type — and Forge fixes that mapping at load time, so it can never be swapped for a different renderer class when a god changes form later. The workaround: `GodEntityRenderer` lazily builds and holds a single `GodHumanoidGeoRenderer` instance internally, and calls its `render()` method directly whenever `dw_form=humanoid`, rather than ever registering it as its own renderer. GeckoLib's animation state lives on the entity's own `AnimatableInstanceCache`, not on the renderer, so this delegation is safe — the same entity object renders correctly either way depending on which method gets called for it that frame. `"god"` form renders nothing here at all (the puppet is invisible; the real boss body is a separate, normal entity everyone already sees), and `"disguise"` form falls back to a plain vanilla `PlayerModel` with a Steve/Alex skin chosen randomly at disguise-entry and cached in NBT.

`GodEntityManager` (client-side) used to actually spawn entities into the client `Level` — fixed (`B-01`) once it became clear that's illegal in Forge: a client-spawned entity is invisible to the server, and every ability/damage method that checks `!level.isClientSide()` would silently no-op on it. It's now a pure tracker (which god type the local player is) plus a thin dispatch: `executeGodAbility()` just forwards to whatever `IGodEntity` the *server* already spawned into the world. `IGodEntity` itself is a small interface (`useAbility`, `toggleFlight`, `addPlayerInventory`, plus two defaulted getters) every god's true-form entity class implements — `AICreakingEntity` (the Creaking's actual GeckoLib boss body, registered as `divineworld:ai_creaking`) is one concrete example, with its own documented animation-to-ability mapping (`walk`/`run`/`tentacle_run`/`tentacles_wall_climb` on a looping base controller, matching the true-form animation names already listed in [[blockbench-assets]]).

`GodAbilityVisualHandler` is unrelated to any of the above — a simple `LivingHurtEvent` listener that spawns 10 god-type-specific particles (dragon breath, smoke, sculk soul, bubbles, spores, or enchant glint) plus a matching sound for 3 of the 6 types, purely as melee-hit juice, whenever a god-tagged player damages something.