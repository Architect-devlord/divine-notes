---
type: system
status: ingested
---

# Crafting System

💡 **What this is**: `CraftCommand.java` + `CraftingWalkManager.java` — an entirely new system (added in the "Added submodules for the animations... added commands to make the agents craft and breed" commit) with zero prior documentation anywhere on this wiki. Teaches early agents the crafting mechanic by demonstration, using the exact same design pattern [[breeding-system]]'s guided-walk addition uses.

## "Control overridden, not consciousness overridden"

This is the actual design principle, stated directly in the source: while a crafting session is in progress, the system forces the agent's position/inventory each server tick — but the perception pipeline (WebSocket stream, [[observation-space]]'s obs builder, [[cognitive-loop]]) keeps running completely normally throughout. The agent isn't driving its own body during the demonstration, but it's watching and remembering every step of it — described in-source as "it can see what's happening to it but can't stop it." This is a teaching mechanic, not a cutscene: the experience genuinely lands in the agent's memory the same way any self-directed action would.

## `/craft minecraft:<item> <agent_name>`

Sequential validation, stopping at the first failure:

1. The item string resolves to a real Minecraft item (accepts both `minecraft:stone` and bare `stone`).
2. The player name resolves to an online AI agent — a real (non-agent) player gets a distinct error from "not found at all."
3. The recipe exists in the vanilla/Forge recipe book for that item.
4. The agent's own inventory actually contains every required ingredient, in the required quantities — ingredient groups are totaled correctly even when a group (e.g. "4 planks") can be satisfied by any matching candidate item, not just one specific item.
5. Routes to one of two `CraftingWalkManager` modes based on the recipe's grid requirements.

## Two modes

- **`INVENTORY`** — recipe is shapeless, or shaped and fits 2×2 (every player's own inventory grid). No walk at all — `startInventoryCraft()` executes on the same tick `/craft` was run.
- **`TABLE`** — recipe needs the 3×3 grid. `CraftCommand` searches a 20-block radius for a crafting table (same search idiom `BreedCommand` uses for beds — see [[breeding-system]]), `AStarPathfinder` finds a route to it, and `CraftingWalkManager.startTableCraft()` walks the agent there before crafting.

`CraftingWalkManager` itself runs a per-agent tick-based session state machine — `Phase.WALKING → ARRIVED → DONE`/`FAILED` — advancing along the precomputed path each tick until arrival, then executing the craft.

## A version-pinning note worth preserving

The source carries an explicit warning that this file targets Forge **1.20.1** specifically: `RecipeHolder<T>` (the ID+Recipe wrapper) wasn't introduced until 1.20.5, so in 1.20.1 `RecipeManager.getAllRecipesFor()` returns `List<T>` directly (unwrapped) and `Recipe.getId()`/`getResultItem()`/`assemble()` take a plain `RegistryAccess`, not the 1.20.5+ `HolderLookup.Provider`. If this project ever upgrades past 1.20.4, this file's recipe lookups will need the `RecipeHolder` unwrap added back — check the target version's migration docs first, don't assume the current code is forward-compatible.

## Shared infrastructure

`AStarPathfinder.java` (`com.divineworld.utils`) — a standalone grid-based A* implementation, built specifically because vanilla `PathNavigation`/`WalkNodeEvaluator` requires a live `Mob` reference, and DivineWorld agents are `ServerPlayer` instances with no `PathNavigation` at all. 8-directional plus single-block step-up/down, bounded by `maxRadius`/`maxNodes` so a worst-case search can't hang the server tick thread (it runs synchronously on the calling thread). Returns `null` on failure — callers decide their own fallback. Shared by both this system and [[breeding-system]]'s guided walk.

## Tick-by-tick internals, now read in full

Runs on `TickEvent.ServerTickEvent`, `Phase.END` only, iterating a static `ACTIVE` session list — same tick phase and "last write wins" idea [[breeding-system]]'s walk manager and `GodControlHandler`'s boss-body sync both use, so a session's forced position always has final say for that tick regardless of what the normal AI pipeline computed. `WALK_SPEED_PER_TICK = 0.22` (blocks/tick, ≈4.4 blocks/sec) and `WAYPOINT_EPSILON = 0.3` — identical constants to [[breeding-system]]'s walk manager, and the same `AStarPathfinder.findPath(level, start, goal, 28, 4000)` call signature (28-block search radius, 4000-node cap).

Each tick in `WALKING`: take the current path waypoint, compute the vector to it, and if closer than `WAYPOINT_EPSILON` advance to the next waypoint (or flip to `ARRIVED` if that was the last one); otherwise step `min(1.0, WALK_SPEED_PER_TICK / dist)` of the way there via `player.moveTo()`, facing the yaw computed from `atan2(-dx, dz)`. For `TABLE` mode specifically, every tick also re-checks that the target block is still a real `CraftingTableBlock` — if the table gets broken mid-walk, the session fails immediately rather than arriving at an empty space. `ARRIVED` double-checks distance against `TABLE_ARRIVAL_RADIUS = 2.0` (a guard for a path that technically ends slightly short — nudges back to `WALKING` for one more step rather than failing) before actually crafting and marking `DONE`.

The craft itself (`executeCraft`) is server-authoritative inventory manipulation, not a simulated GUI click: it re-collapses `slotAssignments` into a per-item-key required count (mirroring `CraftCommand.buildIngredientMap`'s own collapsing, so a shared ingredient across multiple recipe slots isn't double-removed), shrinks matching stacks by exactly that count, then adds the result — dropping it as a world item entity if the inventory is full rather than discarding it silently.

`INVENTORY` mode skips all of the above — `startInventoryCraft()` calls `executeCraft()` directly on the same tick, no `Session`/pathfinding/`ACTIVE` entry involved at all.
