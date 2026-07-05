---
type: system
status: ingested
---
# Memory & BrainCapsule

💡 **What this is**: three distinct storage systems that are easy to conflate but serve different purposes — general event memory, RL replay buffer, and the actual portable "identity" file.

## UnifiedMemoryStore — general event memory

Constructed per-agent (`agent_id`, `capacity=10000`, optional ScyllaDB backend via `use_scylla`/`scylla_hosts`). Every kind of event gets `remember()`-ed here with tags: chat messages, visual/audio observations, file uploads, web pages browsed, autonomous speech, learning experiences. Retrieval via `recall(n)` (most recent), `search(query, limit)`, and `get_training_batch(batch_size, tags)` for the language-learning worker. `.events` is a plain accessible list — `len(agent.memory.events)` is used directly in several places rather than going through a method.

ScyllaDB is the intended production backend specifically because of write volume — see [[hardware-requirements]]: many agents writing memory events continuously needs NVMe-backed storage, not spinning disk.

## EpisodicMemory — the RL replay buffer

A **separate** structure from `UnifiedMemoryStore`, specifically for reinforcement learning: `add(obs, action, reward, next_obs, done, priority)`, capacity 50,000. This was a real bug source — `EpisodicMemory` was never actually initialized in earlier versions, so `learn()`'s `hasattr` guard was always false and the PPO-style replay buffer stayed empty for an entire live session regardless of how long the agent ran. Now initialized eagerly in `NPCAgent.__init__`. On save, capped to the most recent 10,000 transitions (~5MB) regardless of how long the run actually was, specifically to bound file size.

## BrainCapsule — the actual portable identity

This is the file that makes the design wiki's "temporary body, continuous identity" idea literally true in code. Saved as `brain.pcap`. Contents, per `agent.save()`:

**`BrainCapsule` object itself:**
- `metadata` — agent_id, custom_name, agent_type, mode, gender, step_count, saved_at, autonomous flag, `god_type` (a fixed bug: previously not persisted at all, so a restarted god agent lost its ability space and 18-dim policy), and the resolved TCP/backend ports.
- `personality` — full trait dict, see [[personality-and-emotion]].
- `emotion_snapshot` — full emotion dict at save time.
- `memory_snapshot` — up to 1000 most recent memory events.
- `language_state` — the language system's own `state_dict()`, if language was initialized.
- `gender`, `pregnancy_state` — pregnancy state pulled from the breeding system if the agent has one in progress (see [[breeding-system]]).

**Separate `model_state` dict** (neural weights, kept apart from the "identity" fields above):
- Policy weights (`TransformerPolicy` or `GodTransformerPolicy`).
- World model config + weights.
- Vision vocabulary state.
- RewardSystem's RND and ICM network weights.
- Up to 10,000 `EpisodicMemory` transitions + priorities.
- `WorldModelTrainer`'s step count and loss history (so training resumes from the correct step rather than restarting at 0 — also a fixed bug).

## Save cadence

See [[agent-runtime]] for the full detail and a flagged discrepancy with the design wiki's stated target: the actual code saves every **300 seconds** during a live run, once at startup before the server accepts connections, and once on shutdown.

## Confirms the design wiki's claim

[[wiki/design/Robots|Robots]] → BrainCapsule Architecture describes personality, emotional state, memory, knowledge/skills, and goals as the stored contents. The actual `BrainCapsule` dataclass matches this closely — personality and emotion are genuinely separate fields (not merged), exactly as the design wiki describes them: "personality is the slow-moving baseline; emotion is the fast-moving state layered on top of it."
