---
type: reference
status: ingested
---

# Blockbench Assets

💡 **What this is**: the actual 3-D model/animation/texture files under `Blockbench_models/` that [[forms-and-disguise]] documents the _code_ for. That page covers `GodHumanoidGeoModel`/`CreakingGeoModel`/etc.; this page covers what those classes actually load.

## One folder per body, two forms each

`Creaking/`, `Elder_Guardian/`, `Ender_Dragon/`, `Oracle/`, `Warden/`, `Wither/` each hold two complete, parallel asset sets — confirmed by opening one pair directly (Creaking's):

| | Humanoid form | True form |
|---|---|---|
| Source | `God_Creaking.bbmodel` (72K) | `Creaking.bbmodel` (268K) |
| Geometry | `god_creaking.geo.json` (9 bones: `body`/`head`/`torso`/`leftArm`/`leftItem`/`rightArm`/`rightItem`/`leftLeg`/`rightLeg` — a standard bipedal rig, identical shape to every other god's humanoid form) | `creaking.geo.json` (21 bones: `Head2`/`Body2`/arms/legs plus 12 numbered `tentacle` bones and a `jaw`/`mouth` pair) |
| Animations | `god_creaking.animation.json` (16K — `idle`/`walk`/`run`/`swim`/`sneak`/`hit`/`mount`/`eating`/`shield`/`bow`/`crossbow`: humanoid, item-holding actions) | `creaking.animation.json` (80K — `tentacles_out`/`tentacles_retract`/`tentacle_attack`/`grab_eat`/`tentacle_wall_climb`/`burrow`/`dig_out`: creature-specific) |

This lines up exactly with [[forms-and-disguise]]'s code-level description — the humanoid form's uniform 9-bone rig matches `GodHumanoidGeoModel<T>` being one shared class across all six gods, while each true form is a dedicated, much larger rig matching its own dedicated `*GeoModel`/`*GeoRenderer` pair. The size gap (a plain bipedal model vs. a 12-tentacle creature) is exactly why the humanoid path could be shared code and the true-form path couldn't.

Format is GeckoLib (`format_version: 1.12.0` in the exported `.geo.json`, `meta.model_format: geckolib_model` in the `.bbmodel` source) — the standard Forge/Fabric animated-model pipeline, authored in the Blockbench editor and exported to the `.geo.json`/`.animation.json` pair the Java renderer classes load at runtime. Textures are 64×64 in every case checked.

**Naming note**: [[forms-and-disguise]] describes the true form's Java-side assets as `ai_creaking.*` — the source files here are just `creaking.*` (no `ai_` prefix). Most likely the `Blockbench_models/` folder is the authoring workspace and gets renamed/copied into the actual mod resource pack at build time, rather than being loaded from this exact path — not confirmed directly this pass, since the mod's own resource folder wasn't cross-checked byte-for-byte against this one.

## `Human/` — the Civilian Agent, not a shared god template

`BaseHumanoid.bbmodel` + `humanoid.animation.json` + `Humanoid(steve).png`, no `.geo.json` present and no per-god variants. Reads as the [[wiki/design/Robots/Normal_Agents|Civilian Agent]]'s own dedicated body — a plain player-skin-style humanoid — rather than a shared base the six gods' humanoid forms are derived from (each god's humanoid `.geo.json` is its own separate file, not obviously generated from this one). Not directly confirmed which Java class loads it specifically; inferred from naming (`Human`, a Steve-style skin, no god-specific variant needed since Civilian Agents don't have the dual-form system).

## Two vendored folders, not original assets

`Serious-Player-Animations-Template-Resource-Pack/` and `bedrock-samples/` are the two actual git submodules declared in `divine-world-core`'s `.gitmodules` (external repos — a third-party animation template pack and Mojang's own Bedrock sample assets), sitting alongside the god folders as reference material rather than Divine World's own content. Not documented further here — they're upstream projects, not this codebase.
