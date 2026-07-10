---
type: reference
status: ingested
---
# God Abilities

💡 **What this is**: `ai_core/god_controls.py`'s `GodControlSystem` — the actual per-god ability tables and how they're encoded into the 18-dim god action space referenced throughout [[agent-runtime]] and [[reward-and-learning-stack]]. This page is the concrete reference; those pages cover the surrounding architecture.

## Design rule

Ability outcomes are evaluated through the agent's normal `RewardSystem`, tagged `god_ability` — there's no separate reward function for abilities. `integrate_god_controls()` does **not** monkey-patch `agent.act()`; it adds a clean `agent.use_god_ability(name, **params)` API that the cognitive loop or planner calls explicitly, the same deliberate pattern as every other decision in this codebase.

## The six ability sets

As of the latest ingest (two abilities added to two gods since this page was first written — see the fix writeup below):

|God|Abilities (cooldown)|
|---|---|
|**Ender Dragon** (`ender_dragon`/`dragon`)|dragon_breath (5s) · fireball (2s) · perch (0s) · fly (0s) · transform (5s) · revert (1s)|
|**Wither**|wither_skull (1s) · dash (3s) · summon_wither_skeletons (10s) · explosion (8s) · fly (0s) · transform (5s) · revert (1s)|
|**Warden**|sonic_boom (5s) · darkness (15s) · sniff (0s) · burrow (10s) · emerge (0s) · transform (5s) · revert (1s)|
|**Oracle**|wisdom_aura (30s) · foresight (5s) · teleport (3s) · healing_wave (10s) · knowledge_beam (4s) · fly (0s) · **summon_vexes (10s)** · **summon_fangs (4s)** · transform (5s) · revert (1s)|
|**Elder Guardian**|mining_fatigue (60s) · laser_beam (3s) · thorn_attack (5s) · guardian_spikes (20s) · transform (5s) · revert (1s)|
|**Creaking**|toggle_underground (1s) · toggle_ceiling (1s) · deploy_tentacles (0.5s) · life_steal (2s) · tentacle_whip (0.8s) · **retract_tentacles (2s)** · **emerge (0s)** · transform (5s) · revert (1s)|

Every god carries `transform`/`revert` — a shared disguise mechanic, not specific to any one god (see [[forms-and-disguise]]). `fly` is shared by every _flight-capable_ god (Ender Dragon, Wither, Warden, Oracle) but notably absent from Elder Guardian and Creaking. **Partially resolved, 2026-07-05** (see [[wiki/codebase/files/god_controls|god_controls.py]] for the full account): `transform`/`revert` _are_ selectable via the same `ability_idx` mechanism as every other ability — `encode_ability_to_action()`/`decode_action()` index into `GodControlSystem.ability_names()`, which does include them, not into `action_format_sync.py`'s list. Their absence from that Java-matching list may be correct-by-design, since `transform`/`revert` are handled by `GodDisguiseHandler` (a different Java class than `ServerGodAbilityExecutor`, which the rest of that list matches). Still genuinely open: whether the Java side that receives an incoming action frame actually special-cases the string `"transform"`/`"revert"` to route to `GodDisguiseHandler` rather than attempting a `ServerGodAbilityExecutor` lookup that would fail to find them — that specific wiring wasn't re-confirmed this pass.

## `action_format_sync.py` — the authoritative Java-index-matching source

Not previously covered on this wiki directly: `py_backend/utils/action_format_sync.py`'s `GOD_ABILITY_NAMES` dict is the canonical list `god_controls.py`'s own `ability_names()` has to stay in lockstep with — its own header comment states this explicitly: _"Index order matches ability_idx dim 14 in the god action array"_ and _"must match `BaseGodEntity` subclass ability lists (`AIWither`, `AIEnderDragon`, `AIWarden`, `AIElderGuardian`, `AIOracle`, `AICreaking`)"_. This is genuinely two independent Python lists (this one, and `god_controls.py`'s dict) that both have to agree with each other _and_ with the Java side — three-way agreement, not two.

## Two newly-added abilities, and why they were unreachable before

**Oracle gained `summon_vexes` (10s) and `summon_fangs` (4s).** Root cause, straight from the fix comment: Oracle's boss body is `EntityType.EVOKER`, but every Mob-type god body gets `setNoAi(true)` unconditionally (`GodSpawnHandler.spawnGodBody()`) — which stops the vanilla Evoker's own Goal-based spellcasting (`SummonSpellGoal`/`EvokerAttackSpell`) from ever running on its own, since Goals require a ticking `goalSelector`. `ServerGodAbilityExecutor.executeOracleAbility()` already had real, working cases for both — the _only_ path that could ever trigger them — but neither name existed in `god_controls.py`'s dict or `action_format_sync.py`'s list, so the trained `ability_idx` output head had no index that could ever select them, regardless of what the policy learned.

**Creaking gained `retract_tentacles` (2s) and `emerge` (0s, gated by the `dw_burrowed` state flag rather than a timer — same convention as `fly`).** Same root cause, and worth understanding why these aren't just "the same as" the abilities already there: `deploy_tentacles` only ever turns tentacles _on_ — `retract_tentacles` is the only thing that turns them _off_. Likewise `toggle_underground`/`burrow` only ever goes _down_. The Java code's own comments ("AI controls emerge," "AI sends this when ready to surface") confirm `emerge` was always meant to exist as a deliberate, separate agent decision — it just had nowhere to be selected from. This gap was worse than Oracle's: a burrowed Creaking had **no voluntary way to resurface at all** until this fix — it would stay underground and invulnerable indefinitely, since nothing else in the pipeline called `emerge` on its behalf. (Contrast with Warden, which had `burrow`/`emerge` as a correctly-paired set from the start — Creaking's gap was a real asymmetry against an already-correct pattern elsewhere in the same file, not a novel design question.)

Both additions were appended at the **end** of their respective lists in both files, specifically to avoid reassigning the index of any existing ability — inserting `summon_vexes` alongside the semantically-similar `knowledge_beam`, for instance, would have silently shifted `fly` from index 5 to 6 for Oracle specifically, scrambling the learned index↔ability mapping for any already-trained Oracle policy. This is the append-don't-insert principle from the two historical bugs below, now confirmed a second time by a real fix that got it right.

## Two earlier fixed indexing bugs worth knowing about if you're debugging ability selection

- **Wither**: a `blue_skull` ability existed in this table with no corresponding case in the Java client or `ServerGodAbilityExecutor` — it was silently causing an index mismatch (wither ability index 1 was `blue_skull` here but `dash` in `action_format_sync.py`), so the wrong ability fired on every policy output that selected index 1. Removed.
- **Creaking**: `life_steal` and `tentacle_whip` were defined in the opposite order from `action_format_sync.py`'s index order, causing the same class of wrong-ability-fires-on-this-index bug. Reordered to match.

The general lesson, now demonstrated three times over: this ability table's _order_ is load-bearing (it's how a policy-output index maps to an ability name), so it has to stay in lockstep with the Java-side table encoding the same mapping. A purely additive change (new ability appended at the end) is safe; reordering or removing from the middle isn't, without checking the Java side too.

## Action encoding — the 18-dim god action space

Confirms the exact layout referenced in [[agent-runtime]]: base 13 dims (0–12) + 5 god dims (13–17): `[trigger_flag, ability_idx, param1, param2, param3]`.

- `encode_ability_to_action(base_action, ability_name, params)` — builds the full 18-dim vector. A fixed bug lived here too: the old version concatenated `base_action[:11]` (dropping sprint/hotbar) instead of `base_action[:13]`, and placed the trigger flag at dim 11 (sprint's slot) instead of dim 13 — abilities never actually triggered, because `act_god()` was reading trigger at dim 13 the whole time and finding zero there.
- `decode_action(extended_action)` — the reverse: reads `extended_action[13]` as the trigger flag (must be ≥0.5), `extended_action[14]` as the ability index into `ability_names()`.
- `get_params_from_action()` — pulls `param1`/`param2`/`param3` from dims 15–17.

## `use_god_ability()` — the actual call site

Tries the ability (fails silently — returns `False` — if on cooldown or unknown), and **does not send anything to Minecraft directly**. The ability name/params flow back through the normal `act_god()` return value, which `communication_protocol.py`'s `ActionFrame` builder picks up and puts into the binary frame's `god_ability`/`god_params` fields (see [[communication-protocol]]) — calling a Minecraft send from here too would double-send. When an `outcome` dict is supplied (the mod's response to a previous fire), the outcome is routed through `RewardSystem.compute_reward()`/`apply_signal()` and logged to memory tagged `god_ability` — so the brain genuinely learns from whether an ability actually landed, not just from firing it.

Newly integrated abilities are also registered as planner action templates (`agent.planner.add_template({'type': 'god_ability', 'ability': name})`), so deliberation (see [[brain-core]], [[imagination-and-planning]]) considers them alongside ordinary actions rather than abilities living in a separate decision path.