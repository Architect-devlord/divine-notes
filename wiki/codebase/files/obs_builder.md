---
type: file
status: ingested
---
# obs_builder.py

💡 **Role**: the exact 128-dim observation vector every neural component in this codebase (`world_model.py`, `rl/policy.py`, `rl/env.py`) actually consumes — one canonical layout, built here once, so "what does dimension 47 mean" has exactly one answer across the whole codebase.

## Imports

`numpy`, `typing` — external. No internal project imports — deliberately standalone, matching [[wiki/codebase/files/brain_capsule|brain_capsule.py]]'s same design choice, so it can be imported from anywhere (`ai_core`, `rl`) without pulling in the rest of `ai_core`.

## Module-level constant

- `OBS_DIM = 128` — the single source of truth every neural component's input layer sizes against.

## Classes

### `ObservationBuilder`

**Class constant**: `EMOTION_KEYS = ['joy', 'fear', 'anger', 'sadness', 'curiosity', 'trust', 'surprise', 'frustration', 'disgust', 'anticipation']` — ten entries. **Flagged, not previously documented**: [[wiki/codebase/files/emotion|emotion.py]]'s `EmotionSystem.EMOTIONS` has only eight entries and doesn't include `curiosity` or `frustration` at all. Since `build()` reads each key via `.get(key, 0.0)` against an `EmotionSystem.snapshot()`, the two observation-vector slots for `curiosity`/`frustration` are permanently zero — two of the 128 carefully-budgeted dimensions are dead weight given the current `EmotionSystem` implementation. Not confirmed as intentional (e.g. reserved for a planned future emotion) or a genuine mismatch introduced when one file changed without the other — worth a direct question to Devlord rather than guessing further.

**Methods:**

- `build(agent_state)` — the actual layout, by index range (documented in the class's own docstring and confirmed by reading the method): `[0:4]` vitals (health/hunger/saturation/air, normalized); `[4:7]` position (world XYZ, each divided by a fixed 1000-block scale — the same normalization convention [[wiki/codebase/files/brain_language|brain_language.py]]'s `ContextSchema` independently uses); `[7:10]` velocity; `[10:12]` orientation (yaw/pitch, normalized to `[-1, 1]`); `[12:13]` light level; `[13:14]` time-of-day; `[14:15]` is-raining flag; `[15:42]` a 3×3×3 nearby-block-type neighborhood (27 slots — matches [[perception-and-actuation]]/[[wiki/codebase/files/communication_protocol|communication_protocol.py]]'s `nearby_blocks` field exactly); `[42:78]` hotbar contents (9 slots × 4 values: item-type-hash normalized, count, durability, is-selected); `[78:118]` up to 8 nearby entities × 5 values (type-hash, relative x/y/z, distance) — padded with zeros if fewer than 8 are visible after [[wiki/codebase/files/los_filter|los_filter.py]]'s filtering; `[118:128]` the ten emotion values above.
- `build_batch(agent_states)` — vectorized `build()` over a list, stacked into one `(N, 128)` array.
- `get_feature_names()` — a flat list of 128 human-readable labels matching the layout above, for debugging/visualization tooling.

No async methods, no other classes, no module-level functions besides the constant.

## Design commitment, stated directly in the file's own module docstring

"No labels, only physics" — every slot is a directly-measurable physical or game-state quantity (position, velocity, block IDs, item hashes), never a hand-labeled semantic category. The same anti-pretrained-labeling philosophy [[wiki/codebase/files/vision|vision.py]] applies to visual perception and [[wiki/codebase/files/brain_language|brain_language.py]] applies to language, applied here to the full observation vector as a whole.

## Problems (faced by traditional AI systems / LLMs)

Any codebase with more than one place that could plausibly construct "the observation vector" risks each one drifting to a slightly different layout — exactly the kind of mismatch that produced the stale `action_dim=11` findings across `world_model.py`/`vision.py` earlier in this ingest, applied to the observation side instead of the action side.

## Solutions

One canonical builder, imported by every consumer rather than reimplemented, is the direct answer — though as the flagged `EMOTION_KEYS` mismatch above shows, a single canonical builder still isn't immune to drifting out of sync with a _different_ file (`emotion.py`) it depends on for one of its own sections.

## Files Required

None imported directly — but its `EMOTION_KEYS` constant assumes a shape from [[wiki/codebase/files/emotion|emotion.py]]'s `EmotionSystem` that doesn't currently match (see above).

## Files Used In

- `world_model.py` — `_build_observation_from_context()` and the transformer's expected input width both key off `OBS_DIM` (see [[wiki/codebase/files/world_model|world_model.py]] and [[world-model]]).
- `rl/policy.py` / `rl/env.py` — both size their observation space to `OBS_DIM` directly (see [[wiki/codebase/files/policy|policy.py]] and [[wiki/codebase/files/env|env.py]]).