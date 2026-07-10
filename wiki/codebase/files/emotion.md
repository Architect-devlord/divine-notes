---
type: file
status: ingested
---
# emotion.py

💡 **Role**: the smallest core file in `ai_core/` — eight decaying emotion values, nothing more. Every reward-producing file in this codebase (`reward_system.py`, `audio_processors.py`, `brain_language.py`) reads or writes through this class rather than each maintaining its own emotional state.

## Imports

`numpy`, `typing`, `collections.deque` — all external/stdlib. No internal project imports.

## Classes

### `EmotionSystem`

**Class constant**: `EMOTIONS = ['joy', 'sadness', 'anger', 'fear', 'trust', 'surprise', 'anticipation', 'disgust']` — eight entries. **Worth knowing, not previously flagged**: [[wiki/codebase/files/obs_builder|obs_builder.py]]'s `EMOTION_KEYS` list has ten entries, including `curiosity` and `frustration` — neither of which exists here. Those two observation-vector slots read `.get(key, 0.0)` against a snapshot this class never produces those keys for, so they're permanently zero. See [[wiki/codebase/files/obs_builder|obs_builder.py]] for the fuller account.

**Methods:**

- `__init__()` — all eight emotions start at 0.0; a `deque(maxlen=50)` records history.
- `add(emotion, delta)` — clips to `[-1, 1]` after adding; silently ignores an unrecognized emotion name rather than raising.
- `decay(rate=0.05)` — every emotion moves `rate` fraction of the way back toward 0 each call — the mechanism keeping emotional state transient rather than permanently accumulating.
- `snapshot()` — a plain dict copy — what every caller elsewhere in the codebase actually reads.
- `dominant_emotion()` — the single highest-magnitude emotion by absolute value, with its actual (signed) value.
- `get_mood_descriptor()` — a coarse three-way label (`positive`/`negative`/`neutral`) from the sum of positive emotions (joy/trust/anticipation) vs. negative ones (sadness/anger/fear/disgust) — `surprise` counts toward neither side.

No async methods, no other classes, no module-level functions.

## Problems (faced by traditional AI systems / LLMs)

Emotional state in an agent architecture is easy to over-engineer (continuous appraisal theories, full OCC models) or under-engineer (a single scalar "mood"). Both extremes make it hard for the rest of a codebase to reason about: too complex, and every caller needs to understand the whole model to read one value; too simple, and there's nothing for `reward_system.py`'s per-emotion trait-weighted deltas to actually hook into.

## Solutions

Eight named, independently-decaying scalars is a middle ground: rich enough that [[reward_system]] can target specific emotions with specific trait-weighted formulas (fear scaled by neuroticism, trust by agreeableness, and so on), simple enough that any caller just reads `snapshot()` and gets a plain dict. Decay-toward-zero rather than a fixed decay target means a sustained pattern of the same emotion (which `reward_system.py`'s `_apply_drift()` separately tracks) is what actually reshapes personality over time — a single value's transient spike doesn't.

## Files Required

None.

## Files Used In

- `agent.py` — constructs `EmotionSystem` (see [[agent]] and [[agent-runtime]]).
- `reward_system.py` — the primary consumer; every `_emotion_deltas()`/`_apply_drift()` call reads and writes through this class (see [[reward_system]]).
- `brain_core.py` — `_reward_to_emotion_delta()` fallback path (see [[brain_core]]).
- `audio_processors.py` / `brain_language.py` — both feed detected/generated emotion labels back through this system rather than maintaining their own (see [[audio_processors]] and [[wiki/codebase/files/brain_language|brain_language.py]]).