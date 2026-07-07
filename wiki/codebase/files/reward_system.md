---
type: file
status: ingested
---
# reward_system.py

💡 **Role**: turns any agent experience into a personality-weighted scalar, plus emotion deltas and a slow personality nudge — every event in the codebase that produces reward goes through here. [[reward-and-learning-stack]] already covers RND/ICM at a glance, `familiarity_r`, `evidence_r`, breeding reward, and the eager-init fix — this page's real additions: `ChannelNormalizer` (not mentioned on the topic page at all), the full `compute_reward()`/`apply_signal()` mechanics, and the personality-pressure/drift math underneath both.

## Imports

`numpy`, `torch`, `torch.nn`, `collections` (`deque`, `defaultdict`), `typing`, `logging` — all external. No internal project imports; this file is deliberately decoupled from specific subsystems (see `_extract_domain()` below) and only assumes its `personality`/`emotion_system` constructor arguments expose a handful of methods.

## Module-level functions

- `sycophancy_weight(traits)` — personality-derived anti-sycophancy pressure: `0.1 + 0.3 * max(0, conscientiousness − agreeableness)`. Shared with `brain_language.py`, which wraps this exact function rather than maintaining its own copy — see [[language-system]].
- `_extract_domain(url)` — minimal domain extraction for the evidence-diversity term, deliberately independent of `web_browser.py`'s own domain-extraction helper since this file has no reference to a live browser instance.

## Classes

### `RandomNetworkDistillation` (`nn.Module`)

Fixed random target network + trainable predictor; `compute_bonus()` is the prediction-error curiosity signal, `update()` trains the predictor toward the (frozen) target. Standard RND.

**Methods:** `compute_bonus(obs)`, `update(obs)`.

### `ICMModule` (`nn.Module`)

Encoder + forward model + inverse model. `compute_bonus()` is forward-model prediction error in latent space; `update()` trains both forward and inverse heads jointly (`loss = forward_loss + 0.2 * inverse_loss`).

**Methods:** `compute_bonus(obs, action, next_obs)`, `update(obs, action, next_obs)`.

### `ActionEntropyTracker`

A capped-window frequency counter over rounded action vectors. `bonus()` rewards rarity: `-log(freq)/10`, clipped to `[0, 1]` — an action never seen in the window scores highest.

**Methods:** `update(action)`, `bonus(action)`.

### `RewardSignal`

A `__slots__`-based structured return value for `compute_reward()` — nine reward channels plus `emotion_deltas`/`personality_pressure` dicts. `to_dict()` for serialization.

### `ChannelNormalizer`

**Not on the topic page.** Per-channel running-std normalization, added specifically so whichever reward channel has the largest natural magnitude doesn't dominate the combined signal regardless of which one is actually most important right now — a curiosity spike from something novel would otherwise silently drown out a small social/familiarity signal. Deliberately std-only, not full z-score: mean-subtracting would make an objectively-positive event read as negative just for being below its channel's recent average, which the file's own comment argues an agent shouldn't be penalized for. A 50-event warm-up passes values through unchanged rather than dividing by an unreliable early std estimate; a channel with near-zero variance (`std < 1e-4`) also passes through rather than dividing by (near) zero.

**Methods:** `normalize(channel, value)`, `normalize_all(channels)`, `get_stds()`.

### `RewardSystem`

The class everything else calls. Constructed once per agent with `obs_dim`, `action_dim`, `personality`, `emotion_system`.

**Methods:**

- `compute_reward(event, obs, action, next_obs, outcome)` — the ~180-line core. Computes seven raw channels (curiosity from RND+ICM; exploration from action entropy; survival from health/hunger/death deltas — deliberately _not_ run through `ChannelNormalizer`, since it has a meaningful absolute scale that normalizing would erase; task; social; familiarity — see [[reward-and-learning-stack]] for why this one exists at all; aesthetic; evidence), normalizes everything except survival through `ChannelNormalizer`, applies joy/fear emotion-modulation to the normalized positive sum and the raw survival term separately, then computes emotion deltas and personality pressure from the _unnormalized_ raw values (the emotion system responds to what physically happened, not a normalized representation of it). Every 200 calls (`DRIFT_EVERY`), triggers `_apply_drift()`.
- `apply_signal(signal)` — the single place emotion deltas and personality pressure actually get applied after a `compute_reward()` call; callers must never touch `emotion_system`/`personality` directly (stated as a hard rule in the file's own module docstring).
- `_emotion_deltas(...)` — maps the seven raw channels into named emotion deltas (joy/sadness/fear/anger/trust/anticipation/surprise/disgust), each gated by a minimum threshold and scaled by relevant traits (neuroticism amplifies fear, agreeableness amplifies trust, openness amplifies joy-from-aesthetic).
- `_personality_pressure(...)` — small, immediate per-event trait nudges (`PRESSURE_LR = 0.0005`) — e.g. sustained trust nudges `extraversion` up, the positive counterpart to the drift-only path that previously could only push it down.
- `_accumulate(ed)` — folds one event's emotion deltas into a running per-emotion sum, feeding the next drift step.
- `_apply_drift()` — every `DRIFT_EVERY` signals, converts the accumulated emotion sums into a slow (`DRIFT_RATE = 0.002`) personality-wide update — the "sustained emotion reshapes personality" mechanism described in the class's own docstring.
- `recent_average(n)` / `emotional_state_label()` — a four-bucket mood label (`satisfied`/`content`/`struggling`/`distressed`) from the last-20 reward average.
- `save(path)` / `load(path)` — RND/ICM network weights only; everything else on this class is either derived or cheap to recompute.

No async methods anywhere in this file.

## Problems (faced by traditional AI systems / LLMs)

Multi-component reward functions have a well-known failure mode: whichever term happens to have the largest natural numeric range dominates the combined signal by construction, regardless of which term the designer actually intended to matter most in a given moment — and that dominance can shift unpredictably as the agent's situation changes. A second, more specific problem: personality traits that only ever decay (nothing pushes them back up) turn into one-way ratchets over a long-lived agent's lifetime, which stops looking like a personality and starts looking like monotonic drift toward one extreme.

## Solutions

`ChannelNormalizer` is the general answer to the first problem — express every channel in its own "surprise units" before combining, so trait weights (not raw magnitude) are the actual arbiter of importance. The extraversion ratchet gets a narrower, symmetric fix: sustained trust (itself partly fed by the familiarity mechanism [[reward-and-learning-stack]] describes) now nudges extraversion up in both `_personality_pressure()` and `_apply_drift()`, mirroring the existing sustained-fear-lowers-boldness and sustained-sadness-lowers-extraversion paths that were already there.

## Files Required

None from within this codebase — this file only assumes duck-typed `personality`/`emotion_system` objects passed into its constructor, never imports their concrete classes.

## Files Used In

- `agent.py` — `initialize_reward_system()` constructs this eagerly at agent startup (see [[agent-runtime]] and [[agent]]).
- `brain_core.py` — `set_reward_system()`, `_reward_to_emotion_delta()` fallback when unattached (see [[brain_core]]).
- `brain_language.py` — reuses `sycophancy_weight()` directly rather than duplicating the formula.
- `cognitive_loop.py` — `_execute_web_browsing()` supplies `claim_support_delta` into a `web_browsed` event, the mechanism behind `evidence_r` (see [[cognitive_loop]]).
- `breeding_system.py` — routes breeding reward through this same system (not yet given its own file-level page — see [[breeding-system]]).