# Index

Catalog of everything in **Divine_Notes** — the whole Divine World project in one place: design concepts and actual implementation, cross-linked to each other. Read this first when answering any query; it's faster than opening every page blind. See also `Wiki.base` for a dynamic, filterable table view of the same content.

This vault merges what used to be two separate wikis (a design-doc vault and a codebase-doc folder living inside the `divine-world-core` repo) into one, specifically so the concept side and the implementation side can reference each other directly instead of through workarounds.

---

## Hub

- [[wiki/design/Robots|Robots]] — the master design doc. Three-layer reality system, Civilian Agent / God Agent terminology, physical-robot compute architecture, BrainCapsule contents, home base standards. Start here for the _concept_.
- [[wiki/codebase/architecture-overview|architecture-overview]] — the four physical pieces of the actual software (Java server mod, Java client mod, Python backend, Electron frontend) and how they talk to each other. Start here for the _implementation_.

## Design — Robot Bodies

**Civilian Agent**: [[wiki/design/Robots/Normal_Agents|Normal_Agents]] — general-purpose humanoid body.

**God Agent** (6 specialized bodies): [[wiki/design/Robots/Creaking|Creaking]] (perception/climbing scout) · [[wiki/design/Robots/Elder_Guardian|Elder_Guardian]] (amphibious) · [[wiki/design/Robots/Warden|Warden]] (durability + echolocation) · [[wiki/design/Robots/Ender_Dragon|Ender_Dragon]] (biomimetic flight) · [[wiki/design/Robots/Wither|Wither]] (vectored-thrust flight) · [[wiki/design/Robots/Oracle|Oracle]] (collective-perception coordinator — **read [[wiki/codebase/oracle-two-systems|oracle-two-systems]] too, there's a second, unrelated "Oracle" in the actual code**).

## Design — Bases

[[wiki/design/Bases/Common_Agent_Base|Common_Agent_Base]] (reference implementation) · [[wiki/design/Bases/Creaking_Observatory|Creaking_Observatory]] · [[wiki/design/Bases/Elder_Guardian_Temple|Elder_Guardian_Temple]] · [[wiki/design/Bases/Warden_Fortress|Warden_Fortress]] · [[wiki/design/Bases/Ender_Dragon_Sanctuary|Ender_Dragon_Sanctuary]] · [[wiki/design/Bases/Wither_Citadel|Wither_Citadel]] · [[wiki/design/Bases/Oracle_Archive|Oracle_Archive]].

## Codebase — Ingested (29 pages, full depth)

- [[wiki/codebase/architecture-overview|architecture-overview]] — the four pieces and how they connect.
- [[wiki/codebase/agent-runtime|agent-runtime]] — `NPCAgent`, the class every agent actually is.
- [[wiki/codebase/cognitive-loop|cognitive-loop]] — perceive→think→reflect→decide→act, deliberation gating, focus-task resolution.
- [[wiki/codebase/brain-core|brain-core]] — `BrainCore`: the fast/slow evaluation split, `should_deliberate`/`should_browse` gates, `DeliberationResult`.
- [[wiki/codebase/imagination-and-planning|imagination-and-planning]] — `CognitivePlanner`/`ImagineResult`, the swappable risk-personality `scoring_fn`, "the agent imagines its own death and chooses freely."
- [[wiki/codebase/world-model|world-model]] — the Transformer architecture, `EnsembleWorldModel`'s uncertainty mechanism, `imagine()`, and the Mental Matrix backend hiding in the same file.
- [[wiki/codebase/observation-space|observation-space]] — the exact 128-dim vector, the "physics only, no semantic labels" philosophy, and the honest limits of its line-of-sight filtering.
- [[wiki/codebase/perception-and-actuation|perception-and-actuation]] — vision/audio/actuators/web-browsing; confirms the sensor/actuator layer already runs across Minecraft, Isaac Sim, _and_ physical-robot backends.
- [[wiki/codebase/rl-training-harness|rl-training-harness]] — the offline PPO/Stable-Baselines3 harness, and how it relates to the always-on learning in reward-and-learning-stack.
- [[wiki/codebase/isaac-sim-integration|isaac-sim-integration]] — the real Isaac Sim 5.0 bridge, and a folder-watcher that auto-loads a `brain.pcap` into a live sim within 5 seconds — concrete confirmation of the design wiki's Layer 2 concept.
- [[wiki/codebase/language-system|language-system]] — `LanguageIntelligence`, the 4-stage progression, online BPE tokenizer, grounding transformer.
- [[wiki/codebase/god-abilities|god-abilities]] — full per-god ability tables (now including 4 newly-discovered-and-fixed abilities for Oracle/Creaking), the 18-dim encoding, three fixed indexing bugs, and `action_format_sync.py` as the authoritative Java-index source.
- [[wiki/codebase/forms-and-disguise|forms-and-disguise]] — the humanoid/true-form rendering split behind `transform`/`revert`, plus the separate arbitrary-mob disguise system it's easy to confuse it with.
- [[wiki/codebase/personality-and-emotion|personality-and-emotion]] — 8 continuous traits, Plutchik's 8 emotions, where they actually get used.
- [[wiki/codebase/memory-and-braincapsule|memory-and-braincapsule]] — three storage systems, what the portable identity file actually contains.
- [[wiki/codebase/reward-and-learning-stack|reward-and-learning-stack]] — RewardSystem (RND/ICM/familiarity/evidence), PolicyBridge, SelfSupervisedTrainer, SkillTracker, ContinualLearner, GRPO.
- [[wiki/codebase/breeding-system|breeding-system]] — reproduction mechanics **and** the new guided `/breed` walk-teaching command.
- [[wiki/codebase/crafting-system|crafting-system]] — the `/craft` command and its "control overridden, not consciousness overridden" teaching pattern.
- [[wiki/codebase/oracle-two-systems|oracle-two-systems]] — **read before touching anything Oracle-related.**
- [[wiki/codebase/communication-protocol|communication-protocol]] — the wire format, and the "no pretrained labels" philosophy.
- [[wiki/codebase/hardware-requirements|hardware-requirements]] — real load profile and machine-sizing for the Minecraft-layer server.
- [[wiki/codebase/known-issues|known-issues]] — synthesis of `Problems.md`/`AUDIT_REPORT.md`/`FIXES_APPLIED.md`, plus two discrepancies this wiki itself surfaced.
- [[wiki/codebase/electron-frontend|electron-frontend]] — the `dw-chat-ui` React/Vite app: chat, mental workspace, world-model graph, cognitive stream — and the confirmation that its Mental Matrix simulator is *not* wired to the backend system of the same name.
- [[wiki/codebase/human-controller-debug|human-controller-debug]] — the standalone human-piloting pipeline-check tool, easy to confuse with Electron's own "Controller Mode" by name alone; includes Devlord's Mode 1/Mode 2 split for its "overrides an agent's controls" behavior.
- [[wiki/codebase/blockbench-assets|blockbench-assets]] — the actual 3-D model/animation/texture files for all six gods' humanoid and true forms, plus the Civilian Agent's own body.
- [[wiki/codebase/commands|commands]] — every server `/command` not already covered on [[breeding-system]]/[[crafting-system]]'s own pages: the admin/genesis surface, `/godtoggle`'s three arities, `/dw npc`, and the Oracle's full Ollama-lifecycle command tree.
- [[wiki/codebase/events|events]] — the three Forge event listeners not already covered elsewhere: server join/leave/tick, client world-join/respawn/logout, and chat forwarding into the agent's perception.
- [[wiki/codebase/utils-and-infrastructure|utils-and-infrastructure]] — agent tagging/detection (the actual source of truth `DWNPCManager`/`DWEventHandler` both defer to), the genesis/divine-reset ritual and the crash it used to cause, in-world documentation books, and the shared particle-circle renderer.
- [[wiki/codebase/client-sensing-and-control|client-sensing-and-control]] — the Minecraft-client half of perception/actuation: vision/audio capture (three old-hardware fixes worth knowing about) and the bit-packed action-execution protocol shared with [[human-controller-debug]].

## Codebase — fully swept

Every item from the previous "not yet ingested" list is now covered: the Electron frontend and the human-controller debug tool (with the Mental Matrix frontend/backend disconnection confirmed and cross-linked both ways), the Mental Matrix classes themselves in `world_model.py` (confirmed hardcoded/deterministic, not actually "powered by world model predictions" despite the class's own docstring), the Blockbench asset files, `CraftingWalkManager`/`BreedingWalkManager`'s tick-by-tick internals, the `GodDisguiseHandler`↔`TransformationHandler` wiring (turned out to be the wrong method in the original guess — `applyGodForm()`, not `applyTransform()`), the Isaac Sim `brain.pcap` round-trip (fully automatic both ways, confirmed), and the rest of the Minecraft mod layer's commands/events/utils.

**Deliberately left at survey depth, not deep-read**: `DWMod.java`/`DWClientMod.java` (mod entry points), `ClientSetup.java`/`DivineClientSetup.java`, and both `ModEntities.java`/`EntityAttributeRegistrar(ation).java` pairs — pure Forge registration boilerplate (entity types, attributes, bus registration) rather than behavior. Flagging their existence here rather than silently treating the sweep as exhaustive, per this vault's accuracy-over-completeness rule — worth a pass if a future question actually hinges on mod-loading order or registration timing specifically.

**New since the sweep above was compiled** (commit `8839776`, one ahead of the `68cd2a7` this sweep was run against — see `log.md`'s [2026-07-05] entry for full detail): a brand-new file `emergent_templates.py`, plus real changes to `brain_core.py`, `cognitive_loop.py`, `planner.py`, and `continual_learner.py` (the template-ceiling/`EmergentSkillPool` fix). These are slated to become the first pages under a new `wiki/codebase/files/` per-file layer — format specified in `demo-features-explaination.md`, not yet folded into the topic-level pages above or into `CLAUDE.md`. Per Devlord, this stays parked until the sweep above was complete — it now is.

## Deprecated — do not re-ingest as current

- `py_backend/oracle_stuff/` — see [[wiki/codebase/oracle-two-systems|oracle-two-systems]]. Confirmed outdated by Devlord.
- `docs/` folder — confirmed broken/outdated by Devlord, excluded from this wiki's scope entirely.

## Raw Notes

Devlord's own original, unedited brainstorm files — **do not link to these individually from anywhere in `wiki/`.** Go through **[[raw/index|Raw Notes Index]]** instead. This is deliberate: it lets Devlord reorganize `raw/` freely without anything in `wiki/` breaking. See `CLAUDE.md` for the full rule.

## Retired / Out of Scope

Earlier drafts leaned on a fuller Minecraft-society framing — tribes, families, professions, religion, settlements. Deliberately retired (see [[wiki/design/Robots|Robots]], the note under Layer 1) — never backed by more than flavor text. Reproduction/breeding survived the trim because it's real and implemented — see [[wiki/codebase/breeding-system|breeding-system]].

`py_backend/oracle_stuff/` and the `docs/` folder in `divine-world-core` are both confirmed outdated by Devlord and excluded from this wiki's scope — see [[wiki/codebase/oracle-two-systems|oracle-two-systems]] and [[wiki/codebase/known-issues|known-issues]].

## Next Steps

- Real build logs once hardware is sourced (weight/thrust numbers vs. the estimates in [[wiki/design/Robots/Ender_Dragon|Ender_Dragon]] / [[wiki/design/Robots/Wither|Wither]]).
- Confirm the `wiki/codebase/files/` per-file format with Devlord, then ingest the five files from commit `8839776` — `emergent_templates.py` first (see the note above).
- Mode 1/Mode 2 split for the human-controller tool (see [[wiki/codebase/human-controller-debug|human-controller-debug]]) — design direction only, not yet implemented.
- A lint pass across both halves — this is now a good point for one, with the codebase side freshly swept end to end.