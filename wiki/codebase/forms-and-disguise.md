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

Tracks a **3-way form cycle** distinct from ordinary mob-morphing: packets prefixed `"form:"` (`form:god` | `form:humanoid` | `form:disguise`) are routed to a separate particle/state path (`FORM_STATE`/`FORM_PARTICLE_TICKS`) from ordinary morph packets (`MORPH_STATE`/`PARTICLE_TICKS`), specifically so a form-change is never misread as "the god morphed into a vanilla mob literally named 'god' or 'humanoid'" by the existing `isMorphed()`/`getMorphType()` logic. Not fully confirmed this pass: the exact wiring between `GodDisguiseHandler.applyTransform()` and the `form:disguise` broadcast — reasonable to assume they're connected given the naming, but worth verifying directly before stating it as fact.

## Not yet ingested

The Blockbench asset pipeline itself (the `.bbmodel`/`.geo.json`/`.animation.json` files added for all six gods plus a base humanoid) wasn't opened this pass — this page covers the _code_ that consumes those assets, not the assets' own content or how they were authored.