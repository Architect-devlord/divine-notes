---
type: file
status: ingested
---

# emergent_templates.py

💡 **Role**: the action-space half of "grow it, don't hand-author it" — the newest file in `ai_core/`, added in commit `8839776`. One class, `EmergentSkillPool`, that lets [[brain-core]]'s `deliberate()` consider action templates nobody wrote by hand.

## Imports

External only — this file has no internal project dependencies: `logging`, `threading`, `typing` (`Any`/`Dict`/`List`), `numpy`.

## Classes

### `EmergentSkillPool`
Incrementally clusters *executed* action vectors (real points in action space, not one-hot categories) weighted by the reward each one earned, and promotes a cluster into a candidate template once it's both common enough and better than the agent's own running-average reward. Deliberately mirrors `vision.py`'s `OnlineVisualVocabulary` — same online-clustering-then-optional-naming shape, applied to actions instead of percepts.

**Methods:**
- `n_skills` (property) — current cluster count.
- `observe(action, reward)` — the one call site that matters: assigns an executed action to its nearest cluster (or starts a new one), nudges that cluster's centre toward it, updates the running reward baseline, and checks for graduation. Returns which skill token the action landed on.
- `drain_newly_graduated()` — pops and returns every skill that's graduated since the last call, as template dicts ready for `add_template()`. Meant to be called once per learning batch, not per observation.
- `graduated_templates()` — all currently-graduated templates, not just new ones since the last drain.
- `name_of(token)` — a token's human name if it has one, else a placeholder like `skill_7`.
- `assign_name(token, name)` — optional, later labelling; never required for a skill to be usable.
- `get_stats()` — skill count, graduated count, total observations, running reward mean, and the name map — the health-check surface mentioned as worth watching in `demo-features-explaination.md`.
- `state_dict()` / `load_state_dict(state)` — full serialization of cluster state, so a pool survives a save/load cycle intact.
- `_mean_reward(token)` — a single cluster's average reward so far.
- `_mean_intra_dist()` — average pairwise distance across a random sample of existing centres, used only as the "is this new point far enough away to deserve its own cluster" threshold.
- `_new_cluster(action, reward)` — appends a fresh, ungraduated cluster.
- `_maybe_graduate(token)` — the actual graduation gate: enough observations *and* mean reward above the running average by at least `reward_margin`.
- `_as_template(token)` — formats one cluster as the `{type, action_vector, discovered, skill_token}` dict shape `deliberate()` and `generate_plan()` expect.

No async methods, and no functions outside the class — everything here is a plain, synchronous, thread-locked (`threading.Lock`) data structure.

## Problems (faced by traditional AI systems / LLMs)

A hand-authored discrete action menu is a hard ceiling on training signal, not just on runtime behavior: if every candidate a policy is ever scored against comes from a fixed enum, gradient-based training (GRPO here) can only ever push the policy toward one of those fixed points, no matter what the raw policy itself discovers works. This is the same shape of problem unsupervised skill-discovery methods (DIAYN, DADS) and Voyager's ever-growing code-skill library both exist to solve — let the space of "things worth doing" grow from experience instead of from a designer's enum.

## Solutions

Cluster what the agent actually *did*, not a label a human chose in advance, and grade graduation against the agent's own lifetime average rather than a fixed constant — so the bar rises and falls with the agent's own experience instead of being tuned once and left stale. Keeping this file free of any dependency on `brain_core.py`/`planner.py` is itself a design choice: it only ever hands finished template dicts to the existing `add_template()` extension point, so discovery can be swapped out or unit-tested completely on its own (see `demo-features-explaination.md`'s synthetic 4,000-step convergence test).

## Files Required

None from within this codebase — see Imports above; every dependency is an external library.

## Files Used In

- [[continual_learner]] — owns the pool instance; `learn_from_buffer()` feeds it every batch's real `(action, reward)` pairs, then drains new graduates.
- Transitively, `planner.py` (`CognitivePlanner.add_template()` is where a graduated skill actually joins the template list — see [[planner]]) and `brain_core.py` (`deliberate()` is what reads that list and gives graduated skills a chance to be scored at all — see [[brain_core]]).
