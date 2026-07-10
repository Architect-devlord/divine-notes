---
type: file
status: ingested
---

# god_controls.py

💡 **Role**: `GodControlSystem` and `integrate_god_controls()` — already covered in real depth on [[god-abilities]], including the per-god ability tables, the Oracle/Creaking indexing fixes, and an explicitly flagged open question about `transform`/`revert`. This page is the mechanical catalog, plus a partial resolution of that open question.

## Imports

`numpy`, `time`, `logging`, `typing`, `dataclasses` (`dataclass`, `field`) — external/stdlib. `from copy import deepcopy` — lazy, inside `__init__()` only. No internal project imports.

## Classes

### `GodAbility` (dataclass)

`name`, `cooldown`, `last_used`. `is_available()` / `seconds_until_ready()` / `mark_used()` — a plain cooldown timer, nothing more.

### `GodControlSystem`

**Class constant**: `_ABILITY_DEFS` — the full six-god ability dictionary [[god-abilities]] already documents per-god; not re-listed here. Two fix comments recorded directly in this dict, both already covered on that page: the Wither's removed `blue_skull` (index-mismatch fix) and the Creaking's `life_steal`/`tentacle_whip` swap (same class of fix).

**Methods:**

- `__init__(god_type)` — `deepcopy()`s the class-level template per instance specifically so cooldown state (`last_used`) is never accidentally shared across two agents of the same god type.
    
- `get_available_abilities()` — `{name: is_ready}` for every ability.
    
- `ability_names()` — `list(self.abilities.keys())`, in dict-insertion order. **This is the resolution to half of [[god-abilities]]'s open question**: this list _does_ include `transform`/`revert` for every god (they're real keys in `_ABILITY_DEFS`), and it's this list — not `action_format_sync.py`'s `GOD_ABILITY_NAMES` — that `encode_ability_to_action()`/`decode_action()` actually index into. So `transform`/`revert` genuinely are selectable via the same `ability_idx` mechanism as every other ability; they were never invisible to the Python-side encoding.
    
    **What's still open, narrowed rather than resolved**: `action_format_sync.py`'s list is specifically described (in its own header, per [[god-abilities]]) as matching `ServerGodAbilityExecutor.java`'s cases — and `transform`/`revert` are handled by a _different_ Java class entirely, `GodDisguiseHandler`'s `cycleGodForm()`/`applyTransform()` (see [[forms-and-disguise]]), not `ServerGodAbilityExecutor`. So the absence from `action_format_sync.py`'s list may be correct-by-design rather than a gap — but whether the Java side that actually receives an incoming action frame's `god_ability` field correctly special-cases the string `"transform"`/`"revert"` to route to `GodDisguiseHandler` instead of attempting (and silently failing) a `ServerGodAbilityExecutor.execute()` lookup wasn't confirmed this pass — that code wasn't re-read here.
    
- `try_use(ability_name, **params)` — cooldown/existence check, returns a plain command dict (`{"god_ability": name, **params}`) or `None`. Explicitly does not send anything anywhere — per its own docstring, sending and reward-routing are both the caller's responsibility.
    
- `encode_ability_to_action(base_action, ability_name, params)` — see [[god-abilities]]/[[agent-runtime]] for the 13+5=18 dim fix this method's own comment documents; concatenates the (now-corrected) full 13-dim base action with a 5-dim god extension built from `ability_names().index(ability_name)`.
    
- `decode_action(extended_action)` — the inverse; reads the same dims 13/14 the encode side writes, returns `None` if the trigger flag is unset or the array is shorter than 18.
    
- `get_params_from_action(extended_action)` — dims 15–17 as `param1`/`param2`/`param3`.
    

## Function

- `integrate_god_controls(agent)` — attaches `agent.god_controls`, sets `agent.god_action_dim = 18`, and defines/attaches `use_god_ability()` as a closure. **Deliberately does not send anything to Minecraft directly** — its own docstring is explicit that the ability name/params returned from `act_god()` are what `communication_protocol.py`'s `ActionFrame` builder actually uses to populate the outgoing binary frame (see [[communication_protocol]]), and that sending here too would double-send. `use_god_ability()`'s reward routing only fires when both `outcome` is provided _and_ `reward_system` is attached — the two-call pattern its own docstring describes (call once to fire, again later with `outcome` once the mod responds) means a caller that never calls back with an outcome gets the ability activated but never scored. Also loops `agent.planner.add_template()` over every ability name at integration time, if a planner is already attached — the mechanism that makes abilities considerable during deliberation at all, not just callable.

No async methods, no other classes.

## Problems (faced by traditional AI systems / LLMs)

An extensible action space (base movement/interaction dims plus a variable per-agent-type ability extension) risks exactly the encoding bugs already documented on [[god-abilities]] and [[agent-runtime]] — a fixed dimension count assumed somewhere that doesn't match the real extended width. A second, narrower problem specific to this file: any ability dict whose iteration order doubles as a trained model's index space makes every future edit to that dict a potential silent regression — inserting a new ability in the "wrong" position shifts every ability after it to a new index, invalidating whatever an already-trained policy learned for those indices.

## Solutions

Already covered on [[god-abilities]]/[[agent-runtime]] for the dimension fixes. For the ordering fragility: every new ability added to `_ABILITY_DEFS` in this file's own history has been appended at the _end_ of its god's dict rather than inserted logically alongside related abilities — visible directly in the Oracle and Creaking fix comments, both of which explain why the new entries sit after `transform`/`revert` rather than grouped with thematically similar abilities. A convention, not an enforced constraint, but a consistent one.

## Files Required

None from within this codebase.

## Files Used In

- `agent.py` — `integrate_god_controls()` is called from `NPCAgent.__init__` for god-type agents (see [[agent]] and [[agent-runtime]]).
- `communication_protocol.py` — `handle_sound_event_websocket()`'s `ability_outcome` message type is what actually delivers the `outcome` dict `use_god_ability()`'s second call needs (see [[communication_protocol]]).
- [[reward_system]] — `compute_reward()`/`apply_signal()` are called directly for every scored ability use.
- `planner.py` — `add_template()` receives every ability name as a candidate template (see [[planner]] and [[imagination-and-planning]]).