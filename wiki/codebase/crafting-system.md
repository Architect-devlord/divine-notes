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

## Not yet ingested

The exact tick-by-tick walk mechanics inside `CraftingWalkManager` (waypoint following, arrival radius handling) weren't read past the class-level structure — worth a follow-up pass alongside a full read of `BreedingWalkManager`, since they're close enough in design that documenting one deeply would likely explain most of the other.
