---
type: file
status: ingested
---
# action_format_sync.py

💡 **Role**: `ActionFormatter` — converts between NPCAgent action arrays (13-dim NPC / 18-dim god) and the `controls` dict `communication_protocol.py`'s `BinaryProtocol` consumes to build a binary `ActionFrame`. Its own docstring is explicit that production actions don't actually flow through this class (`agent.act()` → `ActionFrame` → `pack_action()` directly) — this file exists for testing, logging, and replay. **[[god-abilities]] already covers this file's most important content in real depth** — the `GOD_ABILITY_NAMES` table, its three-way agreement requirement with `god_controls.py` and the Java side, the `summon_vexes`/`summon_fangs`/`retract_tentacles`/`emerge` additions, and the still-open `transform`/`revert` routing question. This page adds the full mechanical catalog of the class itself, which that page doesn't cover, and confirms every ability name and index order in the table below matches [[god-abilities]]'s existing table exactly, cross-checked entry by entry rather than assumed.

## Imports

`numpy` (as `np`), `typing` (`Dict`, `Any`, `Optional`, `List`) — external only.

## Module-level data

- `GOD_ABILITY_NAMES: Dict[str, List[str]]` — six god types, each a name list where position is the actual `ability_idx` a trained policy can select. Confirmed identical, ability-for-ability and in the same order, to [[god-abilities]]'s existing table for all six gods — including the `summon_vexes`/`summon_fangs` and `retract_tentacles`/`emerge` additions already documented there as appended at the end for index-preservation. `transform`/`revert` are absent from every list here, matching what [[god-abilities]] already establishes: they're selected through `god_controls.py`'s separate `ability_names()` instead, not this table — this file's own header comment ("must match `BaseGodEntity` subclass ability lists") is specifically about the `ServerGodAbilityExecutor`-routed abilities, not the disguise mechanic.

## Class

### `ActionFormatter`

All methods are `@staticmethod` — this class is never instantiated, used purely as a namespace.

- `npc_to_controls(action_array)` — 13-dim array → controls dict. Clips to `[-1, 1]` first. `hotbar_slot`: dim 12 > -0.5 maps to slots 0-8 (`(raw + 1.0) / 2.0 * 8.0`, rounded and clamped), otherwise `None` (no change) — the asymmetric range (`-0.5` threshold rather than `0`) is deliberate, giving the policy a wider "don't touch the hotbar" zone than a symmetric split would. Aliased `npc_to_forge` for old call-site compatibility.
- `god_to_controls(action_array, god_type)` — extends the above: calls `npc_to_controls()` for dims 0-12, then reads dim 13 as the trigger flag (`> 0.5`), dim 14 as the ability index. **Carries its own independent copy of the same `DIM-03` fix documented on [[god-abilities]] for `god_controls.py`'s `decode_action()`** — the in-line comment here states the identical root cause (the trained head outputs a raw integer, not a continuous `[-1, 1]` value; the old code applied a range-mapping formula meant for continuous outputs and selected the wrong ability every time) and the identical fix (direct `round()`). Two files, two independent implementations of the same decode step, the same bug existed in both, and both carry matching fix comments — real, confirmed evidence for how tightly `god_controls.py` and this file have to stay in lockstep, not just an assertion. Warns and returns a no-ability controls dict for a too-short array or an unrecognized `god_type`, rather than raising.
- `controls_to_npc(controls)` — the inverse of `npc_to_controls`; `hotbar_slot=None` → sentinel `-1.0`, otherwise the literal inverse of the forward mapping. Aliased `forge_to_npc`.
- `controls_to_god(controls, god_type)` — the inverse of `god_to_controls`; builds on `controls_to_npc()`, looks up `god_ability`'s index via `ability_names.index(...)`, silently produces a no-ability array (`trigger=-1.0`) if the given ability name isn't in the list for that god type rather than raising — consistent with the rest of this file's warn-and-continue posture.
- `controls_to_flags(controls)` — packs the seven boolean action fields into one byte for the binary frame (`jump` MSB → `sprint` LSB); explicitly documented as matching `ActionExecutor.java`'s bit layout on the other side of the wire.
- `validate_array(action_array)` — accepts 13+ dims (18-dim god arrays pass too, as a superset), warns distinctly for the two specific stale shapes this codebase has apparently seen in practice (11-dim: pre-sprint-and-hotbar; 12-dim: sprint added but hotbar not yet) rather than a single generic "wrong length" message — suggesting these two specific stale shapes were real, encountered states worth naming individually, not hypothetical.
- `validate_god_ability(god_ability, god_type)` — checks membership in `GOD_ABILITY_NAMES[god_type]`; its own docstring gives the reason clearly: catching an unrecognized name here, with a loud warning, is better than letting it reach the Java side silently, where an unknown ability string would just be discarded with no feedback at all.

## Problems (faced by traditional AI systems / LLMs)

The specific problem this file's own fix history demonstrates, twice, independently (once here, once in `god_controls.py`, per [[god-abilities]]): a trained policy's discrete output head produces integer choices, but it's easy to write the decode step as if the output were still a continuous, bounded value like the rest of the action vector — the two look identical in the array (just floats) until you check what actually produced them.

## Solutions

`round()`-ing a value that's already known to be an integer output, rather than applying a continuous-range mapping formula to it, is the specific fix — narrow and correct once you know which case you're in, easy to get wrong if the surrounding code (mostly continuous dims) sets the expectation that everything here is a `[-1, 1]` float. The warn-and-continue posture throughout this file (unknown god type, wrong array length, unrecognized ability name) trades strict correctness for resilience — this is a testing/logging/replay utility, not the production path, so a bad input producing a loud warning and a safe no-op default is a more useful failure mode here than an exception would be.

## Files Required

None — `numpy`/`typing` only.

## Files Used In

- Not confirmed as directly imported by any file read so far this session — [[god-abilities]] already establishes that `god_controls.py`'s own `ability_names()`/`encode_ability_to_action()`/`decode_action()` are what production code actually uses, with this file's `GOD_ABILITY_NAMES` serving as the table those methods have to stay in lockstep with, not as a shared import. Consistent with this file's own docstring framing itself as a testing/logging/replay utility rather than something on the production path.