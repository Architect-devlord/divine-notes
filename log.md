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

**Next session should start with**: `brain_core.py` (see `index.md`'s codebase priority list).