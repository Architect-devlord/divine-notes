---
type: file
status: ingested
---
# breeding_system.py

💡 **Role**: the full reproduction lifecycle — eligibility, pregnancy, birth, growth — that [[breeding-system]] already covers conceptually (including the Java-side walk manager and the "role resolution isn't duplicated" point). This page is the Python mechanics underneath: pregnancy state, the exact god-role-flip logic, and trait inheritance, none of which the topic page walks through line-by-line.

## Imports

`time`, `random`, `logging`, `dataclasses` (`dataclass`, `asdict`), `typing`, `collections.deque` — external/stdlib. One internal import: `from ai_core.personality import can_breed, determine_child_gender, assign_npc_gender, GenderType` — confirms the connection [[wiki/codebase/files/personality|personality.py]]'s page flagged as probable but unconfirmed; now confirmed directly.

## Classes

### `PregnancyData` (dataclass)

`female_id`, `male_id`, `conception_time`, `due_time`, `child_traits` (a full trait dict), `child_gender`. `to_dict()`/`from_dict()` — the serialization pair `BreedingSystem.attach_to_agent()` uses to persist and restore a pregnancy across a save/load cycle; `child_gender` is deliberately kept as a plain string on restore rather than re-validated against `GenderType`, since it's a `Literal` type alias, not a real enum, and plain strings already satisfy it.

### `BreedingSystem`

**Methods:**

- `__init__(spawner)` — three dicts: `pregnancies` (keyed by the carrying parent's id), `growth_stages` (child id → birth time), `breeding_cooldowns` (agent id → cooldown-end timestamp). Takes an [[auto_packager]] `EnhancedAgentSpawner` so it can spawn children and look up existing agents by id.
- `attach_to_agent(agent)` — **fixes bug B-06**: must be called for every agent regardless of type, not just NPCs, since the class's own comment is explicit that gods can carry pregnancies too (when acting in the female role — see below) and that state has to survive a save/load cycle the same way an NPC's would. Also where a restored pregnancy gets re-registered: if the due time already passed while the agent was offline, birth is queued to fire on the very next `tick()` rather than being lost.
- `check_can_breed(agent_a_id, agent_b_id, beds_adjacent=True)` — five preconditions, all must pass: bed-adjacency (waived entirely if either party is a god — "gods are divine beings and don't need beds," per the comment), gender compatibility (via [[wiki/codebase/files/personality|personality.py]]'s `can_breed()`), neither already pregnant, neither still a child, neither on cooldown.
- `initiate_breeding(agent_a_id, agent_b_id)` — **the exact role-flip logic** [[breeding-system]] describes only in prose, now the precise rule set: female+non-female → female carries; dual(god)+male → god carries (acts female); dual+female → the NPC carries (god acts male); dual+dual (god×god) → coin flip, so neither god is permanently locked into the pregnant role across repeated pairings. Child traits come from `_generate_child_traits()` below; child gender from `determine_child_gender()`, defensively re-rolled to plain male/female if it somehow returned anything else (the comment calls this guard purely defensive, since the current implementation of `determine_child_gender()` already only ever returns those two). Fires a `'breeding'` reward event through _both_ parents via `_fire_breeding_reward()` — not hardcoded reward, computed by each parent's own `RewardSystem.compute_reward()` so it's personality-dependent per parent.
- `_fire_breeding_reward(agent, partner_id)` — looks for `agent.reward_system` first, falls back to `agent.brain.reward_system` if not found directly, and just logs (doesn't raise) if neither exists — breeding itself is never blocked by a missing reward system, only the reward signal is skipped.
- `tick()` — the only method meant to be called every game cycle; just sequences `update_pregnancies()` then `update_growth()` and returns any births.
- `update_pregnancies()` — checks every pregnancy's due time against `time.time()`, spawns children for any that are due, removes them from the pregnancy dict either way (success or failure — a failed spawn doesn't leave a permanently-stuck pregnancy).
- `_spawn_child(pregnancy)` — **offspring are always NPCs, even from two gods** — divine heritage is expressed purely through inherited trait _values_, never through spawning another god. Child gender is always male/female, defensively re-rolled here too if the stored value is somehow neither. Calls `attach_to_agent()` on the new child immediately, so a child born mid-session can itself become a parent later without needing a separate wiring step.
- `update_growth()` — linear size interpolation, 0.5 → 1.0 over `GROWTH_DAYS` (a module constant, not shown in this file's own read range but referenced throughout); flips `is_child` to `False` and removes the growth-tracking entry once progress reaches 1.0.
- `_generate_child_traits(parent_a, parent_b, mutation_rate=0.15)` — takes the _union_ of both parents' trait keys (not just `Personality.TRAITS`'s fixed eight — future-proofed against new traits being added later), explicitly skips `gender` (handled separately, never inherited as a blended value), blends each shared trait with a random weight (`uniform(0.4, 0.6)`, not always exactly 50/50) plus independent mutation noise per trait, clipped to `[-1, 1]`.
- `get_pregnancy_status(agent_id)` / `get_serialisable_pregnancy(agent_id)` / `get_growth_status(agent_id)` — read-only status queries; the serializable variant deliberately excludes computed/ephemeral fields (`days_remaining`, etc.) that would be stale the moment they're restored from a save, keeping only the stable fields `PregnancyData.to_dict()` provides.

No async methods anywhere in this file.

## Problems (faced by traditional AI systems / LLMs)

A reproduction/inheritance system with more than two possible parent roles (here: male/female/dual) can't reuse simple binary parent-role assignment — something has to decide, consistently and fairly, which of two dual-gender parents carries a pregnancy without ever permanently locking one into that role. A second, more general problem: any stateful system whose state needs to survive a process restart (here: pregnancies, mid-growth children) has to be attached and restored consistently for _every_ entity type that can hold that state, not just the common case — exactly the gap `attach_to_agent()`'s B-06 fix closes.

## Solutions

The role-flip rule set handles every parent-type combination explicitly rather than falling back to a default, and the god×god case specifically uses a coin flip rather than a fixed rule, so repeated breeding between the same two gods doesn't always assign the same one as the pregnant party. The restart-survival problem gets a direct fix: call `attach_to_agent()` unconditionally for every agent type, and check for an overdue pregnancy on restore rather than assuming any pending state is still comfortably in the future.

## Files Required

- [[wiki/codebase/files/personality|personality.py]] — `can_breed()`, `determine_child_gender()`, `assign_npc_gender()`, `GenderType`.

## Files Used In

- [[auto_packager]] — `EnhancedAgentSpawner` is passed into `BreedingSystem.__init__()`; confirmed directly, matching what this page already said, now that [[auto_packager]] has been read.
- `agent.py` — presumably attaches `BreedingSystem` to each agent (matching `attach_to_agent()`'s call pattern) and calls `tick()` periodically; not directly confirmed at that call site this pass.
- [[reward_system]] — `compute_reward()`/`apply_signal()` are called directly for every breeding event, both parents.
- The Java-side `BreedCommand`/`BreedingWalkManager` (see [[breeding-system]]) ultimately trigger this class's `initiate_breeding()` via `BreedingEventHandler`/`PythonBackendClient` — the full cross-language path is already documented there.