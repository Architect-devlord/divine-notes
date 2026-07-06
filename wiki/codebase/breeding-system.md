---
type: system
status: ingested
---

# Breeding System

💡 **What this is**: `py_backend/breeding_system.py` — the one social/reproductive mechanic that survived the design wiki's trim of the broader tribes/families/society framing (see [[wiki/design/Robots|Robots]]'s retirement note under Layer 1). This one's real and implemented, not flavor text.

## Two paths to breeding, not one

Confirmed from the latest commit: there are now genuinely two separate routes into the same underlying mechanics.

- **Ambient** (what the rest of this page describes below) — `BreedingEventHandler.onServerTick()` watches for agents that wander adjacent to beds on their own and breed organically.
- **Guided** — `/breed <agent_a> <agent_b>`, added specifically ("this will help the early agents learn how to breed") for agents that haven't discovered the mechanic organically yet. `BreedCommand.java` validates both names resolve to online agents, reuses `BreedingEventHandler.areGendersCompatible()` (the exact same predicate the ambient path uses — no duplicated compatibility logic), finds an adjacent bed pair within 20 blocks, checks both agents are within range and it's night or a thunderstorm, then hands off to `BreedingWalkManager`.

**Same "control overridden, not consciousness overridden" pattern as [[crafting-system]]** (the two were added in the same commit, and share the underlying `AStarPathfinder`): `BreedingWalkManager` A*-paths both agents to the beds and forces their position/rotation every tick while walking and sleeping — but the perception/WebSocket loop runs completely normally throughout, so both agents genuinely observe and remember the whole sequence. A `dw_breeding_walk_active` NBT flag is set on both agents for the duration so other movement-applying systems can check and defer, though the walk manager's own tick-end override has final say regardless of whether every other system checks that flag.

Critically, **role resolution isn't duplicated in the Java layer** — `BreedCommand`/`BreedingWalkManager` explicitly do not re-implement gender/role assignment (including the dual-god random-role logic). Once both agents are confirmed asleep, `BreedingWalkManager` triggers `BreedingEventHandler.triggerDirectBreeding()`, which itself goes through the same `PythonBackendClient.notifyBreeding()` → `/api/breeding/event` → `initiate_breeding()` path the ambient route already uses. The guided command is a scripted "get them there" layer on top of the existing mechanics, not a parallel implementation of them.

### Tick-by-tick internals, now read in full

Five phases, not [[crafting-system]]'s four: `WALKING → ARRIVED → SLEEPING → DONE`/`FAILED` — both agents tracked in one `Session` (an `AgentProgress` pair), both driven by the same `Phase.END` tick handler. `WALKING` advances each agent independently along its own precomputed path (`advanceAlongPath()` — identical waypoint-epsilon/step-scale/yaw math to [[crafting-system]]'s walk, since both were clearly written from the same template) and only flips to `ARRIVED` once *both* have reached their final waypoint.

`ARRIVED` re-checks `level.isDay()` — night or thunderstorm could have ended during the walk, and this is checked again here rather than trusting the check `BreedCommand` did at the start, matching vanilla's own behavior of refusing to let a bed-sleep begin under changed conditions. Both agents then call `startSleepInBed(bedHeadPos)`; only once *both* succeed does the phase advance to `SLEEPING` and `BreedingEventHandler.triggerDirectBreeding()` fire. If only one succeeds — the comment notes this shouldn't normally happen since both just passed the identical day/night check, but flags a possible asymmetric `OBSTRUCTED` edge case — the session simply stays in `ARRIVED` and retries next tick rather than failing outright.

`SLEEPING` holds for a flat `SLEEP_HOLD_TICKS = 60` (3 real seconds — long enough to read the chat message, short enough not to feel stuck) before waking both agents and marking `DONE`. A death check for either agent runs first, every tick, regardless of phase — failing the whole session immediately with a message naming whichever agent died.

---

## Timing (Minecraft-day based)

```
MINECRAFT_DAY_SECONDS   = 1200   # 20 real minutes per MC day
PREGNANCY_DAYS          = 9      # 9 MC days = 3 real hours
GROWTH_DAYS             = 30     # 30 MC days to reach adult size
BREEDING_COOLDOWN_DAYS  = 24     # 24 MC days = 8 real hours
```

Full lifecycle: adjacent-bed detection (a Minecraft mechanic, reused rather than reinvented) → conception → 3-real-hour pregnancy → birth at 0.5× scale → 8-real-hour growth curve to 1.0× adult scale → 8-real-hour cooldown before the same parent can breed again.

`PregnancyData` (a dataclass: `female_id`, `male_id`, `conception_time`, `due_time`, `child_traits`, `child_gender`) is what gets serialized into `pregnancy_state` on the BrainCapsule (see [[memory-and-braincapsule]]) — a pregnancy survives a save/restart cycle correctly.

## Eligibility

From [[personality-and-emotion]]'s `can_breed(a, b)`: a `'dual'`-gender agent (every God Agent, always) can pair with anything; otherwise it's strictly one `'male'` and one `'female'`. Child gender is randomized (`determine_child_gender`), not inherited from either parent.

## Trait inheritance

Child traits inherit from both parents with mutation applied (the actual mutation mechanics live deeper in this file than was read for this page — worth a follow-up ingest pass if the exact blend/mutation formula matters for a specific question).

## Reward design — the interesting part

This is worth reading closely because it's a good example of how this codebase avoids hardcoded rewards in general. Straight from the file's own docstring:

> Successful breeding fires a 'breeding' event through each parent's RewardSystem, not as a hardcoded value. The actual reward is computed by `compute_reward()` and scaled by the agent's own personality: high sociability + agreeableness → stronger reward; high neuroticism → reward partially offset by mild anxiety spike; current joy/trust mood amplifies the reward. Hard cap at 0.15 so breeding never dominates survival signals.

In plain terms: a naturally social, agreeable, currently-happy agent genuinely *wants* to have children. A withdrawn or anxious one is at best indifferent, and might even find it mildly stressful. This is computed per-agent from [[personality-and-emotion]] and [[reward-and-learning-stack]]'s `RewardSystem`, not read from a config value — confirms exactly what the design wiki's [[wiki/design/Robots|Robots]] page claims under "Reproduction & Inheritance."

## Automatic packaging

Newborn children get automatically packaged (see [[architecture-overview]] → Packaging and deployment) — a child isn't a passive record in a parent's memory, it becomes its own real, independently-runnable `NPCAgent` process.
