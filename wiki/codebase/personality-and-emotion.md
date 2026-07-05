---
type: system
status: ingested
---

# Personality & Emotion

💡 **What this is**: `ai_core/personality.py` and `ai_core/emotion.py` — both genuinely small, self-contained files, but they quietly influence almost everything else (cooldown timers, reward shaping, breeding drive, speech thresholds).

## Personality — 8 continuous traits, not a typology

```
TRAITS = ['openness', 'conscientiousness', 'extraversion', 'agreeableness',
          'neuroticism', 'boldness', 'curiosity', 'sociability']
```

Each trait is a float clamped to **[-1.0, 1.0]**, defaulting to **0.0** (neutral) unless explicit values are passed in at construction. This is a continuous vector, not MBTI or any fixed-category system — confirms what the design wiki states in [[wiki/design/Robots|Robots]] → BrainCapsule Architecture.

- `as_array()` / `apply_update(delta, lr=0.05)` — traits can drift via gradient-style updates, clipped back into range each time.
- `similarity(other)` — cosine similarity between two agents' trait vectors.
- **Gender**: `'male' | 'female' | 'dual'`. `assign_npc_gender()` picks male/female at random; `assign_god_gender()` always returns `'dual'` — every God Agent is gender-dual by construction, not by chance.
- **Breeding compatibility** (`can_breed`): `dual` can pair with anything; otherwise requires exactly one male and one female. See [[breeding-system]].

## Emotion — Plutchik's 8 basic emotions

```
EMOTIONS = ['joy', 'sadness', 'anger', 'fear', 'trust', 'surprise', 'anticipation', 'disgust']
```

Same [-1.0, 1.0] range, all starting at 0.0. Key mechanics:

- **Decay**: every tick, every emotion multiplies by `decay_rate` (default 0.95) — emotions fade toward neutral on their own unless actively reinforced. Called once per cognitive cycle (`_update_cognitive_state()`).
- **`intensity()`** — max absolute value across all 8; used directly as part of the cognitive loop's urgency calculation.
- **`valence()`** — net positive/negative tone: `(joy + trust + anticipation) − (sadness + anger + fear + disgust)`, normalized to [-1, 1].
- **`dominant_emotion()`** — whichever has the largest absolute value; shown in agent status and used to color speech generation context.

## Where these actually get used

Not just cosmetic — both systems feed real behavioral thresholds:

- `CognitiveLoop._update_thresholds()` derives `speech_cooldown`, `action_cooldown`, and `learning_interval` directly from `extraversion`, `sociability`, `boldness`, and `openness`. A high-extraversion, high-sociability agent has a shorter speech cooldown; a bold agent acts faster; a high-openness agent learns more often.
- `RewardSystem` is constructed with a reference to both `personality` and `emotion_system` — see [[reward-and-learning-stack]] and [[breeding-system]] for how they scale reward signals (breeding reward being the clearest example: sociable+agreeable agents want it, high-neuroticism agents get it partially offset by an anxiety spike, current joy/trust amplifies it).
- `generate_internal_thought()`'s decision to actually think out loud is weighted by `(1 − sociability + openness) / 2`.
