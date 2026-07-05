---
type: system
status: ingested
---
# Observation Space

💡 **What this is**: `ai_core/obs_builder.py`'s `build_observation()` — the canonical 128-dim vector every agent's policy and world model actually see. Referenced throughout [[agent-runtime]], [[brain-core]], and [[world-model]] as "the observation"; this page is where it's actually assembled, and it comes with the single clearest statement of this codebase's core design philosophy.

## The governing principle, verbatim from the module docstring

> The agent learns what things are by what they lead to. It does not receive labels, type tags, hostility flags, or social annotations. It receives physics. Reward — via GRPO and the WorldModel's surprise signal — is the only teacher.

This is stated as a rule to actively defend, not just a description — the docstring goes on: _"If you're about to add any of those back, stop."_ Confirms and sharpens what [[communication-protocol]] describes for vision/entities specifically — this is the same commitment applied to the entire observation vector, and it's stricter than it might first sound.

## The 128 dimensions

|Range|Size|Content|
|---|---|---|
|0–7|8|Vitals: health, hunger, oxygen, armor, exp level, fire ticks, is_child, breeding cooldown|
|8–14|7|Position + orientation: x/y/z (wrapped/normalized), yaw, pitch, on_ground, in_water|
|15–22|8|Environment: time of day, moon phase, raining, thundering, light level, biome bucket, temperature, humidity|
|23–49|27|Block neighborhood, 3×3×3 — **passability only** (see below)|
|50–85|36|Inventory: 18 slots × (item_id, count)|
|86–105|20|Nearby entities: 4 × (distance, relative dx/dy/dz, speed) — **physics only**|
|106–115|10|Emotions (Plutchik-adjacent 10-key set, see [[personality-and-emotion]])|
|116–125|10|Memory recency — exponential decay, 60s half-life, one slot per tracked event type|
|126–127|2|Language stage: vocabulary size, stage index|

## What's deliberately _not_ in here, and why each one was removed

This is the part worth actually reading rather than skimming, because each omission is a specific, argued decision:

- **Block identity has no semantic content at all** — not even a type ID. Each of the 27 neighborhood cells is a single 0.0/1.0 passability bit (solid/collidable vs. not), decoded from a Java-side bitmask. No hardness value either — that's meant to be felt through mining _outcomes_, not read off a lookup table. The reasoning given: `vision.py`'s `OnlineVisualVocabulary` (see [[perception-and-actuation]]) already handles block recognition properly, through real online clustering over actually-captured features. A hand-given type ID sitting right next to that would be a cheap shortcut undermining the harder-won self-organizing pathway — and there's a concrete correctness reason too: the old bucket scheme produced values >1.0 for registry IDs past 27, silently breaking the [0,1] normalization every other dimension respects.
- **No `is_hostile` flag on entities.** Distance, relative position, and speed only. Whether something is worth fearing is exactly the kind of thing this architecture wants learned from consequences, not handed over as ground truth.
- **No entity type bucket, no health_norm on entities.** Same reasoning.
- **No social-context block at all** — no `n_agents_nearby`, no `trust_avg`, no `oracle_nearby`. Notably, this means the collective-perception idea in the design wiki's [[wiki/design/Robots/Oracle|Oracle]] page is implemented at the _Oracle's own aggregation layer_, not leaked into every other agent's base observation vector as an ambient "there's a coordinator nearby" signal.
- **`chat_heard` is kept in the memory-recency block, deliberately** — but the docstring is careful about _why_ it's allowed to stay: it's internal bookkeeping of "a vocalization-like pattern occurred recently," not a social label. The agent doesn't "know" it's chat. Meaning arrives later, through reward correlation, same as everything else.

## Graceful degradation

Every field falls back to a neutral default rather than raising if the Java/perception side hasn't populated it yet — explicitly because full-field population from the Minecraft side is "a separate, ongoing effort." The function asserts its own output length (128) at the end, so an index-drift bug (a very easy mistake to make when hand-laying-out fixed dimension ranges like this) fails loudly rather than silently shipping a misaligned vector.

## Line-of-sight filtering — conditional, not guaranteed

Both blocks and entities pass through `los_filter.py` before encoding, but it's worth being precise about what this actually guarantees, since "LOS-filtered" undersells a real design trade-off in the module. Its preferred path trusts an explicit `los`/`in_los`/`visible` boolean if the Java mod's perception payload includes one (a real raycast via `Level.clip()` is cheap and correct on that side, with full world geometry available). If that flag is **absent**, the module does _not_ attempt a geometric guess from the Python side — it passes entities through **unfiltered**. The reasoning, stated directly in the module: the only block data available in Python at that point is the agent's own immediate 3×3×3 neighborhood, far too small to test occlusion beyond a couple of blocks, and a wrong geometric guess would fail _closed_ — silently blinding the agent to something actually in plain sight — which is worse than not filtering at all. So the accurate claim is: entities behind a wall contribute nothing to the observation _when the Java side reports occlusion_; otherwise nothing is filtered, by design, rather than filtered incorrectly. The closest 4 entities (whatever survives filtering, or all of them if it didn't run) are chosen deterministically by distance so the 4 observed slots stay stable frame-to-frame.