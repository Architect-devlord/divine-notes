# Log

Append-only. `grep "^## \[" log.md | tail -5` for the last 5 entries.

---

## [2026-07-02] ingest | Initial 8 sources → design wiki synthesis

Ingested 8 raw brainstorm files (Robots, Normal_Agents, Creaking, Elder_Guardian, Warden, Ender_Dragon, Wither, Oracle) — Devlord's own notes, ranging from a fully-structured doc (Warden) to an unstructured stream-of-consciousness paragraph (Creaking) to a numbered-list sketch (Ender_Dragon).

Synthesis work: converted every source into a consistent structure (💡 Concept intro, 🔹 sectioned body, Key Insight close); cross-checked technical claims against `divine-world-core` rather than inventing specifics; added real component grounding via web search (Blue Robotics thrusters/sonar, gecko dry-adhesive mechanics, the Tel Aviv "Robat" echolocation pattern, ornithopter precedent, gimballed dual-rotor thrust vectoring, Meshtastic LoRa mesh, Hailo AI HAT+); trimmed the tribes/families/professions/religion/settlement framing (kept reproduction/breeding — real and implemented); wrote 7 new base pages that didn't exist before.

Result: 15 design pages, fully cross-linked.

## [2026-07-02] revise | Terminology + compute architecture

Two corrections across all 15 design pages: (1) "NPC" → "Civilian Agent", "god" (category term) → "God Agent" throughout, with a Terminology section disambiguating "Normal Agent" (body) from "Civilian Agent" (mind). (2) Compute architecture flipped — local on-robot compute is now the default, central brain-server is an explicit documented fallback, with concrete latency mitigations. Ender_Dragon/Wither got honest treatment of the resulting weight-budget tension.

## [2026-07-02] setup | Codebase wiki initialized (as a separate structure at the time)

First pass applying the LLM Wiki pattern to `divine-world-core` itself, at the time built as its own `wiki/` folder living inside that repo (not yet merged with the design wiki — see the merge entry below).

## [2026-07-02] ingest | Core cognitive + identity systems (11 codebase pages)

Read, in order: `agent.py`, `cognitive_loop.py`, `personality.py`, `emotion.py`, `breeding_system.py`, `hardware_reqs.txt`, `oracle_stuff/*.py`, the Java Oracle files, `communication_protocol.py` (header + imports), `PythonBackendClient.java`, `Problems.md`, `AUDIT_REPORT.md`, `FIXES_APPLIED.md`. Deliberately did not deep-read everything referenced (`vision.py`, `world_model.py` internals, `brain_core.py` internals, the RL harness, most of the Java mod, the Electron frontend) — recorded honestly in `index.md` rather than claimed as covered.

**Correction applied mid-session**: initially planned to document `oracle_stuff/*.py` as the Oracle God Agent's active training pipeline. Devlord corrected this directly — those scripts are old, outdated, and not part of the current system. [[wiki/codebase/oracle-two-systems|oracle-two-systems]] reflects the correction.

**One discrepancy surfaced** by cross-checking the design wiki against the codebase directly (not from any source doc): the design wiki's save-cadence target (~1 min) doesn't match the actual code (5 min). Recorded in [[wiki/codebase/known-issues|known-issues]] rather than silently resolved either direction.

## [2026-07-03] restructure | Merged into Divine_Notes; raw/ became a self-contained island

Two changes requested by Devlord, both structural:

1. **`raw/` boundary rule.** Previously the design wiki's `index.md`/`log.md`/`CLAUDE.md` referenced `raw/` fairly loosely (described it as the source of the wiki pages, listed originals inline). Devlord flagged these as _his_ temp files, not something the wiki should treat as a stable, linked-into layer. Fixed by: giving `raw/` its own `index.md`, removing every direct `wiki/` → specific-raw-file reference (now `wiki/` only ever points at `raw/index.md`, never a named file inside it), and confirming `raw/` files only link to each other, never out. This also surfaced a real technical issue worth recording: `raw/` and `wiki/design/` share 8 filenames (Robots, Normal_Agents, Creaking, Elder_Guardian, Warden, Ender_Dragon, Wither, Oracle), which makes every bare `[[Name]]` link to one of those ambiguous once both live in the same vault. Fixed by path-qualifying every such link vault-wide (`[[wiki/design/Robots/Creaking|Creaking]]`, `[[raw/Creaking|Creaking]]`, etc.) — 72 link instances across 24 files, verified programmatically post-fix (link resolution, ambiguity, and the raw/ boundary rule all checked, zero violations).
    
2. **Merge + rename.** The design wiki (`DivineWorld-Wiki`) and the codebase wiki (previously living inside `divine-world-core` as a bare `wiki/` folder) are now one vault: `Divine_Notes`. Structure: `raw/` (untouched originals + their own index), `wiki/design/` (the 15 concept pages), `wiki/codebase/` (the 11 implementation pages), one root-level `index.md`/`log.md`/`CLAUDE.md`. The old cross-vault workarounds in the codebase pages (backtick-quoted `` `Robots.md` `` instead of a real link, "separate vault, referenced by description only" caveats) are gone — everything that used to be a plain-text pointer to the other vault is now a real `[[wikilink]]`. Added a few direct pointers between the two halves where they matter most: [[wiki/design/Robots/Oracle|Oracle]] ↔ [[wiki/codebase/oracle-two-systems|oracle-two-systems]], and [[wiki/design/Robots|Robots]]'s Compute Architecture section ↔ [[wiki/codebase/agent-runtime|agent-runtime]]/[[wiki/codebase/hardware-requirements|hardware-requirements]].
    

`Divine_Notes` still lives inside/alongside the `divine-world-core` repo, same placement intent as the old "wiki additions" package — just renamed and now carrying the design content too.

## [2026-07-04] ingest | Brain internals cluster + two new Java gameplay systems

Pulled latest from GitHub first (`git pull` — repo was one commit behind: "Added submodules for the animations and created required minecraft assets for the humanoid form of the gods," which turned out to be much bigger than the name suggests — full Blockbench models/animations/textures for all 6 gods, plus two entirely new commands).

**Python brain internals** (completing the cluster [[wiki/codebase/agent-runtime|agent-runtime]]/[[wiki/codebase/cognitive-loop|cognitive-loop]] already depended on): `brain_core.py` in full → [[wiki/codebase/brain-core|brain-core]]; `planner.py` in full → [[wiki/codebase/imagination-and-planning|imagination-and-planning]]; `god_controls.py` in full → [[wiki/codebase/god-abilities|god-abilities]]; `obs_builder.py` in full → [[wiki/codebase/observation-space|observation-space]]; `world_model.py` (strategic, full class map) → [[wiki/codebase/world-model|world-model]]; `brain_language.py` (strategic) → [[wiki/codebase/language-system|language-system]]; `vision.py`/`audio_processors.py`/`actuators.py`/`web_browser.py` (strategic) → [[wiki/codebase/perception-and-actuation|perception-and-actuation]]. Corrected [[wiki/codebase/reward-and-learning-stack|reward-and-learning-stack]]'s `WorldModelReplayBuffer` capacity claim (was 50,000, actual default is 100,000) and updated its stale "not deep-read" notes now that dedicated pages exist.

**Two new Java gameplay systems**, both added in the same commit and sharing one design principle ("control overridden, not consciousness overridden" — perception keeps running normally while movement/inventory get taken over for a guided demonstration) and one shared utility (`AStarPathfinder.java`, built because DivineWorld agents are `ServerPlayer` instances with no vanilla `PathNavigation`):

- `CraftCommand.java` + `CraftingWalkManager.java` — entirely new, zero prior documentation. → [[wiki/codebase/crafting-system|crafting-system]].
- `BreedCommand.java` + `BreedingWalkManager.java` — a guided-teaching layer on top of the _existing_ ambient breeding system, reusing its compatibility logic and Python entry point rather than duplicating it. → folded into [[wiki/codebase/breeding-system|breeding-system]] as a new section.

**Not ingested this pass, flagged in `index.md`**: the rendering/animation pipeline itself (Blockbench models, `GodHumanoidGeoModel`/`GodHumanoidGeoRenderer`) — six gods went from placeholder to fully modeled/animated/textured in this same commit, and the wiki hasn't caught up to any of it yet; `TransformationHandler.java`/`GodDisguiseHandler.java` (the shared transform/revert ability every god table includes); the Mental Matrix simulation classes discovered living inside `world_model.py`; `los_filter.py`; the RL harness; `isaac_sim_integration.py`; the Electron frontend.

20 codebase pages now, up from 11.

## [2026-07-04] restructure | Synced wiki/design/ to Devlord's manual reorganization

Devlord reorganized `wiki/design/` himself: `Robots.md` stays at the `wiki/design/` root, the 7 body pages moved into a new `Robots/` subfolder, the 7 base pages moved into a new `Bases/` subfolder. Mirrored the same move here and fixed every link that depended on the old flat layout — specifically the 14 path-qualified links (7 robot names + 7 base names) created during the raw/wiki collision fix, which all needed `Robots/` or `Bases/` inserted. `Wiki.base` needed no changes — its views already filter by frontmatter `type` rather than folder depth. Re-verified vault-wide afterward: 333 links, zero broken, zero ambiguous, zero raw/ boundary violations.

## [2026-07-04] ingest | Forms/disguise system + los_filter.py correction

`GodHumanoidGeoModel.java`, `TransformationHandler.java`, `GodDisguiseHandler.java` → [[wiki/codebase/forms-and-disguise|forms-and-disguise]]. Resolves what every god's shared `transform`/`revert` ability (see [[wiki/codebase/god-abilities|god-abilities]]) actually does visually: a humanoid form (one shared renderer class, per-god assets swapped by `god_type`) and a true/boss form (per-god dedicated renderer). Also surfaced a genuinely separate, broader system — `GodDisguiseHandler` lets a god disguise as an arbitrary vanilla mob, not just switch its own two forms — easy to conflate with transform/revert, so the page leads with the distinction.

Also read `los_filter.py` (short, 131 lines) and corrected an overstated claim in [[wiki/codebase/observation-space|observation-space]]: line-of-sight filtering is conditional on the Java side providing an explicit `los` flag, not guaranteed — the Python-side fallback deliberately passes entities through unfiltered rather than guess at occlusion from a too-small 3×3×3 block window, since a wrong guess would fail closed (silently hiding something in plain sight), which the module's own author judged worse than not filtering at all.

21 codebase pages now.

## [2026-07-04] ingest | RL training harness + Isaac Sim bridge

`rl/policy.py` (full), `rl/env.py`/`rl/train.py` (strategic) → [[wiki/codebase/rl-training-harness|rl-training-harness]]. Confirms the 13/18-dim action layout at the actual policy source rather than through callers, and documents the offline-PPO vs. always-on-learning distinction against [[wiki/codebase/reward-and-learning-stack|reward-and-learning-stack]] explicitly, since the two are easy to assume are the same training loop.

`isaac_sim_integration.py` (strategic) → [[wiki/codebase/isaac-sim-integration|isaac-sim-integration]]. Best find of this pass: a real, working folder-watcher that auto-loads a dropped-in `brain.pcap` into a live Isaac Sim session within 5 seconds — concrete, working code behind what the design wiki's Layer 2 concept describes, not just a plausible-sounding plan. Targets Isaac Sim 5.0 (open-sourced SIGGRAPH 2025) with a documented migration from the 4.x API, so this is actively maintained, not a stale stub.

23 codebase pages now, up from 11 at the start of this continued session.

## [2026-07-04] revise | rl/train.py deleted; two gods gained real, previously-unreachable abilities

Devlord pulled ahead of this session twice more while it was in progress:

- **`rl/train.py` deleted as dead code** — nothing imported `train_agent` except its own unused re-export in `rl/__init__.py`, now removed too. [[wiki/codebase/rl-training-harness|rl-training-harness]] and `index.md` both still describe it in past tense (what was read, historically accurate at ingest time) — this entry is the pointer for anyone wondering why a "not yet ingested" list doesn't mention it: it's not pending, it's gone.
- **`god_controls.py` + `action_format_sync.py` both updated in the same upstream commit**: Oracle gained `summon_vexes`/`summon_fangs`, Creaking gained `retract_tentacles`/`emerge` — all four were already fully implemented in `ServerGodAbilityExecutor.java` but unreachable from Python because neither file had an entry for them, so the trained `ability_idx` output head had no index that could select them. Worse for Creaking specifically: a burrowed Creaking had no voluntary way to resurface at all before this fix. Both additions were appended at the end of their lists (not inserted alongside similar abilities) specifically to avoid reassigning any existing ability's index. [[wiki/codebase/god-abilities|god-abilities]] rewritten to cover both, plus the newly-understood role of `action_format_sync.py` as the authoritative Java-index source these two Python files both have to agree with.

**A duplication this session's own context limits caused, worth recording rather than hiding**: partway through, working from an incomplete memory of what this same continued session had already produced, a redundant `rl-and-simulation.md` was created covering the same ground as the already-existing (and more thorough) [[wiki/codebase/rl-training-harness|rl-training-harness]] + [[wiki/codebase/isaac-sim-integration|isaac-sim-integration]]. Caught before delivery, deleted, and the two cross-references that had been added pointing to it were redirected to the correct pages instead. Two near-duplicate log entries and one stale `index.md` section (an old "next ingest candidates" list that had fallen out of sync with the correct one at the bottom of that file) from the same cause were cleaned up in this same pass.

---

## [2026-07-05] lint | Corrected stale next-step pointer; two new items surfaced

**Stale pointer fixed**: this log's closing line said "next session should start with `brain_core.py`" — but that file was already fully ingested back in the [2026-07-04] "Brain internals cluster" entry above, several entries before this line was next read. Recording the correction rather than quietly deleting it, per this vault's own rule about corrections (see the `oracle_stuff` entry, 2026-07-02).

**New upstream commit, not yet ingested**: `divine-world-core` HEAD is now `8839776` ("minor blockbench asset fixes and revising the brain architecture"), one commit ahead of `68cd2a7` (the last commit this wiki accounts for). Real code changed, not just assets: `brain_core.py` (new `_action_to_vector()` helper; `deliberate()` now reads the live `agent.planner.templates` instead of the frozen `DEFAULT_TEMPLATES`), `cognitive_loop.py` (new `PLAN_BLEND` constant — the chosen plan step now nudges the executed action instead of being computed and then ignored), `planner.py` (small — same `_action_to_vector()` treatment in `_run_imagination()`), `continual_learner.py` (now owns a skill pool, fed from `learn_from_buffer()`), and a brand-new file, `emergent_templates.py` (`EmergentSkillPool` — online clustering over *executed* actions, so GRPO's candidate space can grow from lived experience instead of staying permanently capped at the ten hand-authored templates). Also: minor Blockbench `.bbmodel`/`.animation.json` tweaks across all 6 gods (Warden's animation file changed far more than the others — worth a look whenever the Blockbench-asset item finally gets ingested), and `divine-notes` is now a properly registered submodule (`.gitmodules` + pinned commit `5152799`) — matching this vault's own HEAD exactly, so nothing new on that side.

**New documentation format specified, not yet applied anywhere**: `demo-features-explaination.md` (root level) is Devlord's spec for a new per-file layer at `wiki/codebase/files/`, distinct from the existing topic-level `wiki/codebase/*.md` pages. Per-file pages should cover a file's classes/functions; swap "Root Cause" for "problems faced by traditional AI systems/LLMs"; swap "Solution" for "Solutions" (what Divine World actually implements); swap "Files Modified" for "Files Required" (dependencies) / "Files Used In" (dependents); wikilinks throughout, correctly qualified per the collision rule above. Not yet reflected in `CLAUDE.md`'s Structure/Conventions sections. The demo file is itself a real writeup of the commit above, so `emergent_templates.py` is the natural first candidate once the format's confirmed with Devlord.

---

## [2026-07-05] ingest | Electron frontend + human-controller debug tool (25 codebase pages now, up from 23)

Per Devlord: finish the pre-existing "not yet ingested" list before touching the new `wiki/codebase/files/` format or commit `8839776`'s brain-architecture changes — both stay parked (see the entry above) until this list is clear.

**[[wiki/codebase/electron-frontend|electron-frontend]]** (new page) — read the full `dw_agent/electron/react-app` tree (`App.jsx`, `ControllerSafety.jsx`, and all six `components/*.jsx` files). Despite the folder name and the `electron` npm dependency, there's no Electron main-process file or launch script anywhere — it runs today as a plain Vite-served web app. Two dead callback props found (`onSimulationEvent`, `onPredictionRequest` — neither is ever called by the component that accepts it) and one hardcoded-backend-URL inconsistency in `ControllerSafety.jsx`; both recorded in [[known-issues]] rather than only on the page itself.

**Mental Matrix disconnection, confirmed**: Devlord flagged directly that the backend Mental Matrix system (inside `world_model.py`) and the frontend's Mental Matrix simulator/visualizer are *not* connected, and reading both sides independently confirms it — `MentalMatrixSimulator.jsx` makes no backend call at all. [[world-model]]'s earlier "probably feeds the frontend" guess was wrong and has been corrected in place (dated, not silently overwritten).

**[[wiki/codebase/human-controller-debug|human-controller-debug]]** (new page) — read `human_controller_server.py` in full and the relevant two-thirds of `dw_controller.html`. Devlord's clarification, load-bearing for this whole page: this tool's real purpose is pipeline debugging (verify the sensor→action round-trip works, independent of whether the AI stack is even running), *not* a "human roleplays the AI" feature. The current implementation necessarily seizes a real named agent's ports/identity to do this (same ports a real Python agent would bind), which is the "overrides an agent's controls" behavior Devlord flagged as a bug against current intent, though potentially a real feature later. Recorded as a two-mode design direction for future work — Mode 1 (debug/pipeline-check, current intent) vs. Mode 2 (deliberate live takeover, not built) — not yet implemented, just specified.

Also confirmed: this tool is entirely unrelated to `ControllerSafety.jsx`'s "Controller Mode" beyond the name — that one's about the AI gaining device/filesystem/network access, not a human piloting an agent. Both pages now cross-link and flag the collision explicitly.

**Not yet confirmed**: whether `DWClientBot`/`TCPServer.java` treats a human-controller connection differently from a real AI-agent connection at the Minecraft-mod layer — only the Python/browser side was read this pass.

---

## [2026-07-05] ingest | Rest of the Minecraft mod layer — sweep complete (29 codebase pages now, up from 25)

Finished the list from the entry above: Blockbench assets, the Mental Matrix classes inside `world_model.py`, `CraftingWalkManager`/`BreedingWalkManager` internals, the disguise-handler wiring, the Isaac Sim write-back question, and the remaining commands/events/utils layer. In order:

**[[wiki/codebase/blockbench-assets|blockbench-assets]]** (new page) — opened representative `.geo.json`/`.animation.json` pairs for Creaking's humanoid and true forms directly (via Python's `json` module, not just file listing). Confirmed the humanoid/true-form split [[forms-and-disguise]] already documented in code shows up exactly as expected in the assets: a uniform 9-bone rig shared in shape across all six gods' humanoid forms vs. each true form being a dedicated, much larger rig (Creaking's has 21 bones, 12 of them tentacles). `Human/BaseHumanoid.bbmodel` read as the Civilian Agent's own body, not a shared template the six gods derive from — inferred from naming, not directly confirmed against a specific Java loader.

**Mental Matrix classes, `world_model.py` lines 99–810, now fully read** — [[world-model]] updated. The class's own docstring calls it "powered by world model predictions"; nothing in ~700 lines actually calls `WorldModel`/`EnsembleWorldModel`/`forward()`/`imagine()` — hardcoded Euler-integration physics throughout. Also corrected: `register_mental_matrix_api()` actually lives at line 2005, not inside the 99–810 range an earlier pass assumed. The backend's full WS command protocol (`add_object`/`apply_impulse`/`set_running`/etc.) is close enough to what [[electron-frontend]]'s simulator reimplements client-side that it's almost certainly the intended integration point, if that disconnection ever gets fixed.

**`CraftingWalkManager`/`BreedingWalkManager`, both read in full** — folded into [[crafting-system]] and [[breeding-system]] rather than new pages, since the structure was already there. Both share the identical waypoint/yaw/speed math and the same `AStarPathfinder` call signature, confirming they were built from the same template. Breeding's version tracks two agents through an extra `SLEEPING` phase (`SLEEP_HOLD_TICKS = 60`) before handing off to `BreedingEventHandler.triggerDirectBreeding()` — same Python entry point the ambient path already used, no duplicated role-resolution logic in Java.

**`GodDisguiseHandler`↔`TransformationHandler` wiring, confirmed** — [[forms-and-disguise]] corrected. The original phrasing of this item assumed `applyTransform()` produces the `form:disguise` broadcast; it's actually a different method, `applyGodForm()` (part of a separate `god→humanoid→disguise` cycle system with its own `dw_form` NBT key), that does the string concatenation (`"form:" + form`) and sends it. `applyTransform()` handles arbitrary mob-disguises on an unrelated `dw_disguised` key and broadcasts with no prefix at all — two real, independent systems, not one.

**Isaac Sim `brain.pcap` round-trip, confirmed automatic both ways** — [[isaac-sim-integration]] updated. `_save_agent()` writes back to the same path every 10 training episodes, plus once more when training completes. Nothing manual about either direction.

**Rest of the mod layer** — five new pages: [[commands]] (the admin/genesis surface, `/godtoggle`'s three arities, `/dw npc`, the Oracle's 11-subcommand Ollama lifecycle — plus finding `CommandRegistrar` itself used to be dead code, imported but never called), [[events]] (server join/leave/tick and two client handlers — three separate fixed-bug stories live here: double god-registration, ability cooldowns that never ticked down, and a disguise-body sync that stopped following its owner), [[utils-and-infrastructure]] (agent tagging/detection as the actual root of truth, the genesis ritual's fixed client/server crash, book items, the particle-circle renderer), and [[client-sensing-and-control]] (vision/audio capture's three old-hardware fixes, the bit-packed action protocol shared with [[human-controller-debug]], and the morph-sync packet pair). [[forms-and-disguise]] also picked up a new section on the renderer-delegation trick `GodEntityRenderer` uses to work around Forge's fixed renderer registration.

**Deliberately left at survey depth**: `DWMod.java`/`DWClientMod.java`, `ClientSetup.java`/`DivineClientSetup.java`, both `ModEntities.java`/`EntityAttributeRegistrar(ation).java` pairs — pure registration boilerplate, flagged rather than silently skipped.

The `divine-notes` submodule inside `divine-world-core` (empty, uninitialized, pinned to the same commit as this vault's own HEAD) has now been cleaned up per Devlord's instruction to do so once documentation work was done — see below.

---

## [2026-07-05] ingest | First five `wiki/codebase/files/` pages — the format, applied for real

Per Devlord, the sweep above was the prerequisite; this is what came next. All five files from commit `8839776` are done: [[wiki/codebase/files/emergent_templates|emergent_templates.py]], [[wiki/codebase/files/continual_learner|continual_learner.py]], [[wiki/codebase/files/planner|planner.py]], [[wiki/codebase/files/brain_core|brain_core.py]], [[wiki/codebase/files/cognitive_loop|cognitive_loop.py]] — full imports/classes/functions/async-functions catalog, Problems/Solutions framed at the general-AI-systems level rather than this codebase's specific history, Files Required/Files Used In in place of Files Modified. `CLAUDE.md`'s Structure and Conventions sections now formally describe this layer.

**A real linking mistake, made and caught partway through**: Devlord flagged that early drafts of these pages (starting with `brain_core.md`) were linking topic-level page names — `[[agent-runtime]]`, `[[language-system]]`, `[[reward-and-learning-stack]]` — inside Files Required/Files Used In sections as if those names *were* the dependency files. They aren't: the actual files are `agent.py`, `brain_language.py`, and `skill_tracker.py`/`policy_bridge.py`/`self_supervised_trainer.py` respectively, none of which share a name with their topic-level page. Root cause: `wiki/codebase/*.md` topic pages use kebab-case names chosen for readability (`brain-core`, `world-model`), while the new per-file pages are named after real files (`brain_core.py`, `world_model.py`) — similar enough to blur together while writing, especially under `brain_core.md`/`brain-core.md`'s single-character difference. Fixed across all five pages: Files Required/Used In/Imports now name the real file and link to its per-file page only if one exists; if it doesn't yet, the plain filename is used with a separate, explicit pointer to the topic-level page for narrative context, never a link standing in for both at once. Recorded as a formal rule in `CLAUDE.md` rather than just quietly fixed, per this vault's usual practice.

Also applied this pass, per Devlord's parallel note not to trust a comment's claim blindly: cross-checked rather than assumed, e.g. `ClientMorphSyncPacket`'s "packet ID 1" comment against what `ClientNetworkHandler` actually registers (0 — see [[client-sensing-and-control]], found in the previous sweep but the same discipline applied again here), and `_get_dominant_event()`'s assumption that `agent.memory.events` is a plain attribute, confirmed against `agent.py` rather than taken on faith from `cognitive_loop.py`'s own comment.

**Not yet done**: the rest of `py_backend/` (well over a dozen files — `agent.py`, `world_model.py`, `reward_system.py`, `policy_bridge.py`, `skill_tracker.py`, `self_supervised_trainer.py`, `brain_language.py`, `memory.py`, `vision.py`, `audio_processors.py`, `actuators.py`, and more), then the Java mod layer, then the Electron frontend — all still only covered at the topic level. Devlord's own framing: this "will take some time," done incrementally rather than in one pass.

---

**Next session should start with**: continuing the `wiki/codebase/files/` sweep on the Python side — no fixed next file specified; ask Devlord for priority or pick up wherever `py_backend/` coverage is thinnest. Re-check every new page against the linking rule above before considering it done.