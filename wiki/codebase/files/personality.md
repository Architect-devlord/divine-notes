---
type: file
status: ingested
---
# personality.py

💡 **Role**: the trait vector every trait-weighted formula in this codebase reads from (`reward_system.py`'s `sycophancy_weight()`, `_emotion_deltas()`'s per-trait scaling, `cognitive_loop.py`'s cooldown derivation) — plus gender and breeding-compatibility logic, which turns out to live in this file too, not a separate one.

**Correction, 2026-07-05**: an earlier version of this page was wrong in several places — it described a `describe()` method and an `adjust(trait, delta)` method that don't exist in this file at all, and it missed `gender`/`GenderType`/`to_dict()`/`from_dict()`/`apply_update()`/`similarity()` and all four module-level gender/breeding functions entirely. Re-read in full and corrected below rather than patched quietly, per this vault's own rule about recording corrections.

## Imports

`numpy`, `typing` (`Dict`, `Optional`, `Any`, `Literal`) — external. No internal project imports.

## Module-level type

- `GenderType = Literal['male', 'female', 'dual']` — not a class, a type alias. Matches the same three-value scheme the Java-side `TaggedEntitySystem.detectAgentType()` fix (`B-03`, documented on [[breeding-system]]) enforces: gods must carry `'dual'` specifically so either partner in a breeding pair can be treated as either role.

## Classes

### `Personality`

**Class constant**: `TRAITS = ['openness', 'conscientiousness', 'extraversion', 'agreeableness', 'neuroticism', 'boldness', 'curiosity', 'sociability']` — the standard Big Five plus three Divine-World-specific additions.

**Methods:**

- `__init__(gender='male', traits=None)` — **not just traits** — `gender` is a real constructor parameter and `self.gender` a real stored attribute, missed entirely in the earlier version of this page. Any subset of `TRAITS` can be passed via `traits`; anything not given defaults to 0.0, each value clipped to `[-1, 1]` on the way in.
- `to_dict()` — `{'gender': ..., **self.traits}` — gender _and_ all eight traits, not just the traits.
- `from_dict(data)` _(staticmethod)_ — the inverse; pops `gender` out of a copy of the dict first (`data = dict(data)`, explicitly so the caller's own dict is never mutated) before passing the rest as `traits`.
- `as_array()` — the fixed-`TRAITS`-order `numpy` vector other files fold into a larger observation/context vector (see [[wiki/codebase/files/obs_builder|obs_builder.py]] and [[wiki/codebase/files/brain_language|brain_language.py]]). Gender isn't included in this array — only the eight numeric traits.
- `apply_update(delta, lr=0.05)` — **the real method `reward_system.py` actually calls** (confirmed by reading the call sites directly: `_personality_pressure()` and `_apply_drift()` both call this, not a per-trait setter). Takes a full `TRAITS`-length delta array, not a single `(trait, delta)` pair — `as_array() + lr * delta`, clipped to `[-1, 1]`, then written back trait-by-trait.
- `similarity(other)` — cosine similarity between two personalities' `as_array()` vectors — not referenced anywhere in this ingest pass yet; possibly used for matching/compatibility logic not yet traced to a caller.
- `get(trait, default=0.0)` — dict-style single-trait read; this one was correct in the earlier version of this page.
- `__repr__()` — standard debug string.

No `describe()` method — that name doesn't exist in this file.

## Module-level functions (missed entirely in the earlier version of this page)

- `assign_npc_gender()` — random choice of `'male'`/`'female'` (never `'dual'` — that's reserved for gods).
- `assign_god_gender()` — always returns `'dual'`, unconditionally.
- `can_breed(a, b)` — `True` if either gender is `'dual'`, or if the pair is exactly `{'male', 'female'}`. The `'dual'`-is-always-compatible rule is what makes a god able to breed with any NPC regardless of that NPC's own assigned gender.
- `determine_child_gender(a, b)` — random `'male'`/`'female'` choice for the resulting child (never `'dual'` — a child is never spawned as a god).

## Problems (faced by traditional AI systems / LLMs)

A personality model needs to be simple enough for many different call sites across a codebase to each derive their own specific behavior from it (a cooldown, a reward weight, a decision threshold) without every one of those call sites needing custom logic for how personality is stored or accessed. Separately, and unrelated: a breeding system with more than two possible "roles" (here: male/female/dual) needs compatibility rules that don't collapse into simple binary matching.

## Solutions

One flat, named, clipped-range trait vector with both dict-style (`get()`) and array-style (`as_array()`) access covers both common read patterns. `apply_update()` taking a full delta array rather than one trait at a time is what lets [[reward_system]] express drift/pressure as a single vectorized operation instead of a loop over individual trait nudges. The gender/breeding functions solve the compatibility problem with one asymmetric rule (`'dual'` matches anything) rather than needing a full compatibility matrix.

## Files Required

None.

## Files Used In

- `agent.py` — constructs `Personality` from spawn-time `persona_traits` (see [[agent]], [[agent-runtime]], and [[wiki/codebase/files/agent_spawner|agent_spawner.py]] for where those traits are actually chosen).
- [[wiki/codebase/files/agent_spawner|agent_spawner.py]] — imports `GenderType`, `assign_npc_gender()`, `assign_god_gender()` directly from here (**corrected** — an earlier version of that page's Imports section wrongly attributed this to a nonexistent `ai_core.mc_gender` module; fixed there too).
- `reward_system.py` — `_apply_drift()`/`_personality_pressure()` both call `apply_update()`; `sycophancy_weight()` reads `.traits` directly (see [[reward_system]]).
- `cognitive_loop.py` — several methods read `.traits` directly for cooldown/threshold derivation (see [[cognitive_loop]]).
- `obs_builder.py` / `brain_language.py` — both fold `as_array()` into a larger observation/context vector (see [[wiki/codebase/files/obs_builder|obs_builder.py]] and [[wiki/codebase/files/brain_language|brain_language.py]]).
- `breeding_system.py` — `can_breed()`/`determine_child_gender()`/`assign_npc_gender()`/`GenderType` are all imported and used directly (confirmed — see [[wiki/codebase/files/breeding_system|breeding_system.py]]).