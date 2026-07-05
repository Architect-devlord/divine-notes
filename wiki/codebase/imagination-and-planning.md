---
type: system
status: ingested
---

# Imagination & Planning

💡 **What this is**: `ai_core/planner.py` — `CognitivePlanner` and `ImagineResult`. Distinct from [[brain-core]]'s `deliberate()` (which scores single next-actions) — this is multi-step sequence planning, and it comes with a genuinely unusual design commitment worth understanding on its own terms.

## The commitment, straight from the file's header

> The planner gives the agent complete autonomy over its thought space. It can imagine any outcome including its own death, failure, or harm, and uses those trajectories to make decisions. Self-preservation, risk-aversion, or recklessness all emerge from what the agent has LEARNED about consequences — nothing is hardcoded away. The planner imagines faithfully. The agent chooses freely.

This isn't just a comment — it's load-bearing in `ImagineResult` and `imagine_death()` below.

## ImagineResult — nothing is filtered

The full output of imagining one sequence through the world model: per-step rewards, per-step termination (death) probabilities, and the imagined latent states themselves.

- `total_reward` — discounted cumulative reward, γ=0.95.
- `peak_death_probability` — the single worst step in the sequence.
- `expected_survival` — probability of surviving the *entire* sequence: `Π(1 − termination_prob)` across all steps.
- `dies_in_imagination` — `True` if any single step predicts >50% death probability.

None of these are hidden from the scoring function or clipped before the agent sees them. A trajectory where the model predicts near-certain death is reported exactly as such.

## `scoring_fn` — the agent's risk personality, as a swappable function

This is genuinely the most interesting design choice in the file. `CognitivePlanner` takes a `scoring_fn: callable(ImagineResult) -> float` that determines what "good" even means, and the file's own docstring gives real examples of the range this covers:

```python
# Pure reward (default) — values what it learned to value
lambda r: r.total_reward

# Survival-weighted (cautious)
lambda r: r.total_reward * r.expected_survival

# Balanced risk/reward
lambda r: r.total_reward * (0.5 + 0.5 * r.expected_survival)

# Death-curious — seeks dangerous situations specifically to learn from them
lambda r: r.total_reward + r.peak_death_probability * 5

# Pure survival instinct
lambda r: r.expected_survival * 10
```

`set_scoring_fn()` can swap this at runtime — nothing in the architecture says an agent's risk personality has to be fixed for its whole life. Whether anything currently *drives* that swap automatically (tied to personality traits, or triggered by experience) wasn't confirmed in this pass — worth a follow-up read if it matters for a specific question.

## `imagine_death()` — deliberately thinking about what kills you

Samples random sequences and returns them sorted by predicted death probability, highest first. The docstring is explicit about why this exists: *"lets the agent deliberately think about what kills it — essential for building genuine survival intuition from experience rather than having avoidance baked in by the programmer."* This is the same philosophy [[observation-space]] applies to perception (no hardcoded "this is dangerous" flag) applied to planning instead.

## Action templates and encoding

`DEFAULT_TEMPLATES` — 10 seed actions (`collect`, `craft`, `attack`, `flee`, `use_item`, `explore`, `eat`, `build`, plus target/item variants). `ACTION_TYPE_INDEX` — the shared one-hot encoding scheme used identically here, in [[brain-core]]'s deliberation, and in [[god-abilities]]'s ability triggering, so a trained value is comparable across all three call sites. `add_template()` lets new action types register at runtime — the template list isn't fixed at construction.

## `generate_plan()` — the actual multi-step planner

Scores the top 5 templates by single-step value, then samples `n_trials` random sequences built from just those top 5, scoring each with `scoring_fn` (or a table-based fallback sum if no world model). Adds a small novelty bonus for underused action types and a mild repetition penalty for low-diversity sequences, then returns the best sequence found. This feeds directly into [[cognitive-loop]]'s plan-execution-with-interruption logic.
