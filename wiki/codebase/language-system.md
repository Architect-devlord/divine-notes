---
type: system
status: ingested
---

# Language System

💡 **What this is**: `ai_core/brain_language.py`'s `LanguageIntelligence` — how an agent actually learns to talk. Header states the philosophy directly: *"Learns through experience, not pre-training."* Same commitment [[observation-space]] makes for perception, applied to language.

## No pretrained tokenizer, no pretrained vocabulary

`OnlineBPETokenizer` (`max_vocab_size=4096` default) learns its own byte-pair merges from what the agent actually hears/reads — there's no GPT-style pretrained tokenizer anywhere in this path. `Vocabulary` subclasses it directly. This is the language-side equivalent of [[perception-and-actuation]]'s `OnlineVisualVocabulary` — same pattern, different modality.

## Grounding — connecting words to non-linguistic context

`MultimodalGroundingTransformer`: word embeddings + a context encoder (maps a 32-dim context vector into the same concept space) run through a small transformer encoder (2 layers, 4 heads by default), with two output heads — predict the next word, and predict the context a word implies. `ground_word(word_id, context)` returns the concept embedding for a specific word in a specific context — this is the actual mechanism by which a word becomes attached to meaning here, rather than meaning being assumed from a pretrained embedding space.

## Four language stages

```python
names = ['pre-linguistic', 'proto-language', 'linguistic', 'advanced']
```

`language_stage` starts at 0 (`pre-linguistic`) and progresses based on accumulated experience (exact promotion thresholds weren't traced in this pass). Stage gates real behavior, not just cosmetic labeling:

- `should_speak()` returns `False` outright below stage 1.
- `generate_speech()` returns `None` at stage 0.
- Sampling temperature drops from 1.2 to 0.8 at stage ≥2 — a later-stage agent's speech is less random, more confident, by design.

## `should_speak()` — genuinely infrequent by default, not a chat loop

```python
if language_stage < 1: return False
if time_since_last_speech < speech_cooldown: return False
if any emotion's magnitude > 0.6: return True   # strong feeling overrides the roll
sociability = personality.traits.get('sociability', 0.0)
return random() < (sociability + 1) / 40
```

The stochastic roll caps out at 5% per check even for a maximally sociable agent (`sociability=1.0` → `2/40`) — speech is meant to feel occasional and personality-driven, not constant. A strong emotional spike (either direction, magnitude >0.6) bypasses the roll entirely and forces a speech attempt regardless of how introverted the agent's traits are.

## `generate_speech()` — grounded in actual memory, not generic

If a conversation is already active, continues it coherently via `ConversationBuffer` (maxlen 40, tracks `current_topics()`). If not, seeds generation from the agent's own recent memories rather than generating something generic — the intent being that autonomous speech feels like it comes from what the agent has actually experienced, not a disconnected chat-bot utterance.

## `process_input(text, context, speaker_id)`

The main input handler — [[brain-core]]'s `process_language_input()` wraps this directly (and, per a fixed bug noted there, used to be missing the `speaker_id` parameter entirely, which would have thrown on any caller passing it through — now fixed, since `speaker_id` is required for the familiarity/repeat-visitor tracking [[reward-and-learning-stack]]'s `familiarity_r` reward term depends on).

## `learn_from_file(file_path, filetype)`

The mechanism behind an agent learning from a dropped-in document (see [[agent-runtime]]'s `/api/upload` route) — text gets fed through the same experience-based learning path as spoken conversation, not a separate pretraining step.
