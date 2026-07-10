---
type: file
status: ingested
---
# los_filter.py

💡 **Role**: strips entities the agent couldn't actually perceive from a perception frame, purely by geometry — no game-engine raycasting involved. Its conceptual role is already folded into [[observation-space]]'s narrative; this is the mechanical catalog that page doesn't provide.

## Imports

`numpy`, `typing`, `dataclasses`, `math` — all external/stdlib. No internal project imports.

## Classes

### `LineOfSightFilter`

**Methods:**

- `filter_entities(agent_pos, agent_yaw, entities, fov_degrees=110, max_distance=32)` — the actual filter: an entity survives only if it's within `max_distance` _and_ within a horizontal field-of-view cone (`fov_degrees`, split evenly left/right of the agent's yaw). No occlusion/raycasting against terrain — this is a pure distance+angle cone, not "can the agent actually see through walls to this entity."
- `_angle_to(agent_pos, agent_yaw, entity_pos)` — signed horizontal angle between the agent's facing direction and the entity, wrapped to `[-180, 180]`.
- `_in_fov(angle, fov_degrees)` — `abs(angle) <= fov_degrees / 2`.

No async methods, no module-level functions.

## Problems (faced by traditional AI systems / LLMs)

An agent that perceives every entity in a fixed radius regardless of facing direction effectively has omniscient, 360-degree vision — unrealistic for a Minecraft-embodied agent, and it removes a whole category of "did I actually notice this" behavior (turning to look, being surprised by something behind you) that a real field-of-view constraint creates.

## Solutions

A simple, cheap geometric cone (no raycasting against terrain, no visibility graph) gets most of the realism benefit — an agent genuinely can't perceive something directly behind it — at a fraction of the computational cost real occlusion testing would need, appropriate for something recomputed every perception tick.

## Files Required

None.

## Files Used In

- `world_model.py` / `communication_protocol.py`'s `PerceptionFrame` handling — entities are filtered before being folded into the observation the agent's brain actually sees (see [[observation-space]] for the narrative account; not directly traced to a specific call site this pass).