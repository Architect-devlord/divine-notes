
dont make the feature explaination like this this is just a summary of a fix i have used it as demo to make you know about the kind of content in the kind of format that is needed for the explaining the classes and the functions of each of the files and instead of the files modified there should be files required and files used in and then wikilinks should be correctly used . This template explains a fix but you need to explain files for raw. instead of root cause we can explain the problems faced by traditional ai systems , llms in this part and well the solution implemented by us will be written under the solutions tab
# Template Ceiling — deliberate() / GRPO can now learn beyond DEFAULT_TEMPLATES

## Summary
`BrainCore.deliberate()` is what produces `all_scored_actions`, which is the *only* signal
`grpo_update()` trains the policy on. Traced end to end, `deliberate()`'s candidate pool was
unconditionally `planner.py`'s ten hand-authored `DEFAULT_TEMPLATES` for the entire life of every agent — so GRPO could only ever push the policy's continuous output toward one of ten fixed one-hot directions, regardless of what the raw policy discovered on its own. This patch relocates the hardcoding from *what deliberate() is allowed to consider* to *a seed prior it starts from*, and adds a mechanism for the candidate pool to grow from lived experience — mirroring how `vision.py`'s `OnlineVisualVocabulary` already grows its own visual categories with no hand-given labels.

DEFAULT_TEMPLATES is not removed. It's now a bootstrap seed, not a lifetime ceiling.

A companion fix (see "Follow-up fix" below) addresses the separate bug where the plan chosen for
a given tick never influenced the action actually sent to Minecraft — the two are independent and
each stands on its own: this one un-ceilings what GRPO can ever train the policy toward; the other
makes the current tick's plan actually matter, not just future training.

## Root Cause
1. `BrainCore.deliberate()` resolved `candidate_actions` from the frozen module constant
   `ai_core.planner.DEFAULT_TEMPLATES` whenever a caller passed `None` — and both call sites in
   `cognitive_loop.py` (`deliberate(perception)`, no `candidate_actions` argument) always pass `None`.
2. `_deliberate_with_world_model()`, `_deliberate_with_table()`, and `_world_model_action_value()`
   all encoded every candidate as a one-hot vector via `ACTION_TYPE_INDEX.get(candidate['type'], 0)`.
   `all_scored_actions` (score, one-hot vector) pairs are exactly what `grpo_update()`
   (`self_supervised_trainer.py`) fits: `dist = Normal(action_mean, std); loss = -(advantage *
   dist.log_prob(act_tensor)).mean()`. Because `act_tensor` only ever contained these ten basis
   vectors, the gradient had no support anywhere else in action space — this is the actual ceiling,
   not just a conceptual one.
3. `add_template()` — the only existing mechanism that grows the template set (used today only by
   `god_controls.py` for god abilities) — appends to `CognitivePlanner.templates`, a *different*
   list from the frozen `DEFAULT_TEMPLATES` constant `deliberate()` was reading. So even
   runtime-added templates never reached GRPO.
4. `CognitivePlanner.generate_plan()` (which drives `_execute_action()`'s multi-step plan and
   feeds `SkillTracker`'s progress signal) doesn't gate the real Minecraft action — that still comes
   from `agent.decide()` → `policy._predict()`, unconstrained — but it does shape the
   frustration/joy/task-switching signal in `_record_skill_attempt()`, so it was worth fixing too.

## Solution
- **`emergent_templates.py` (new)** — `EmergentSkillPool`, an online k-means clustering pool over
  *executed* action vectors (not one-hot template types), weighted by the reward each one actually
  earned. A cluster only "graduates" into a candidate template once it has enough observations
  *and* a mean reward above the agent's own running average — relative to its own experience, not
  a hand-picked constant. Naming is a separate, optional, later step (`assign_name`), exactly like
  `OnlineVisualVocabulary` — a skill can be tried, scored, and reinforced long before anything calls
  it "collecting wood." Unit-tested standalone: in a synthetic 4,000-step run mixing noise, a
  consistently high-reward motif, and a consistently mediocre-reward motif, exactly one skill
  graduated, and its centroid converged to within 0.09 (L2) of the true high-reward motif while
  staying 1.1 away from the mediocre one.
- **`continual_learner.py`** — `ContinualLearner` now owns an `EmergentSkillPool`. Every call to
  `learn_from_buffer()` feeds the batch's real `(action, reward)` pairs into it, then pushes any
  newly-graduated skills into `agent.planner.templates` via the *existing* `add_template()` — no
  new extension point, reusing the one god_controls.py already established.
- **`brain_core.py`** — `deliberate()`'s candidate resolution now reads `agent.planner.templates`
  (live, growable) instead of re-importing the frozen `DEFAULT_TEMPLATES` constant, falling back to
  it only if the planner or its template list is unavailable. A new shared helper,
  `_action_to_vector()`, replaces the three duplicated one-hot construction blocks: it uses a
  candidate's own `action_vector` when present (discovered skills — a real point in action space,
  not restricted to a basis direction) and falls back to the original `ACTION_TYPE_INDEX` one-hot
  encoding otherwise, so every hand-authored template behaves exactly as before.
- **`planner.py`** — `_run_imagination()` gets the same `_action_to_vector()` treatment, so
  `generate_plan()`'s imagination rollouts don't silently collapse a discovered skill onto whatever
  template happens to occupy one-hot index 0.

## Where "instinct" actually belongs
Human instinct isn't a fixed menu of ten named plans you're allowed to consider — it's a bias on
*which* of effectively infinite actions feel good or bad, plus a few reflexes that bypass
deliberation entirely. This codebase already has that pattern working correctly in
`_deliberate_with_world_model()`: `uncertainty_lambda`, derived from `boldness`/`neuroticism`,
penalizes epistemically-uncertain candidates more or less depending on personality — a scoring
bias, not a restriction on the option set. The fix above follows the same principle: seed
templates and the reward-shaping in `scoring_fn`/`uncertainty_lambda` are where hand-authored
priors should live; the candidate *space* itself should be free to grow. No changes were needed
there — it's already the right pattern, just worth naming so future "instinct" tuning goes there
and not back into a hardcoded list.

## Suggested next step: let instinct evolve across generations, not just within one
`breeding_system.py`'s `_generate_child_traits()` already blends two parents' personality traits
with mutation. The natural extension: seed a newborn's `planner.templates` / skill-pool state the
same way (blend + mutate parents' *graduated* skills), with the raw `DEFAULT_TEMPLATES` constant
becoming the floor only for a first generation or an orphaned spawn — rather than every agent, for
every generation forever, independently starting from the exact same ten. This wasn't implemented
here (it touches `breeding_system.py` and is a separate design decision), but it's a small diff on
top of infrastructure this patch already adds, and it's the most direct way to honor "instincts
evolved too" literally rather than as a metaphor.

## Known Risks / Limitations
- **Cold start is a real tuning problem, not just a footnote.** `min_obs_to_graduate` /
  `reward_margin` too strict → nothing ever graduates and this patch is a no-op. Too loose →
  thrashing gets reinforced as if it were a skill. Start conservative and watch
  `EmergentSkillPool.get_stats()` (mirrors `OnlineVisualVocabulary.get_stats()`) before loosening.
  Quantified (see the section below): the fixed-`lr` EMA update has no outlier rejection, so a
  centroid absorbing 50% off-target noise with no escape converges ~0.45–0.73 away from the true
  motif — but in the actual pool, with splitting enabled, most of that noise gets peeled into its
  own never-graduating clusters instead, and the graduated centroid lands within ~0.04–0.17 in the
  same test. Discrimination between distinct motifs stays strong either way (~0.9 vs ~5.0
  separation, never confused) — this affects precision of a graduated skill's centroid, not
  whether distinct skills get confused for each other.
- **`_table_action_value()` (the no-world-model fallback) doesn't understand discovered skills.**
  It keys `value_table`/`forward_model` by type string, which only ever populated for the ten
  original types — a discovered skill will score 0.0 there, not crash, but it means table-fallback
  mode under-values discovered skills relative to world-model mode. Generalizing the value table to
  arbitrary continuous actions is a bigger, separate design question left open here.
- **`grpo_update()` requires `len(scored_actions) >= 4`.** During early cold-start with very few
  graduated skills this is almost certainly fine (seed templates alone are 10), but worth keeping
  in mind if seed templates are ever reduced.
- **Interpretability regresses until something names a skill.** Logs/dashboards will show
  `skill_17` for a while. `assign_name()` exists for exactly this but nothing calls it yet —
  a natural job for `SkillTracker` or the language system, noticing a correlation between a skill
  token and an item/achievement gained.
- **This changes the literal training signal.** Test on one agent before rolling out to a
  long-running population — this is the kind of change where "it imports cleanly" (verified: all
  four files compile) is necessary but not sufficient evidence it behaves well over a long run.

## Verified since writing this: contamination drift, and how directly this reaches real actions

**Contamination drift is real and now bounded on both ends.** Forcing every observation into a
single cluster (`max_skills=1`, no split escape — isolates the raw EMA update from the pool's
other defenses) and sweeping noise fraction over 2,000 steps / 5 seeds each:

| noise fraction | mean dist to true motif | range |
|---|---|---|
| 0.0  | 0.014 | 0.011–0.016 |
| 0.3  | 0.350 | 0.205–0.529 |
| 0.5  | 0.592 | 0.447–0.728 |
| 0.7  | 0.714 | 0.511–0.900 |

That worst-case number is real: plain fixed-`lr` online k-means has no rejection rule, so a point
that lands nearest an existing cluster gets absorbed regardless of whether it actually belongs
there. With splitting enabled (the actual default configuration, `max_skills=64`), the practical
exposure is smaller — 0.04–0.17 in the equivalent multi-cluster test — because a fair amount of
diffuse noise gets split into its own low-reward clusters that simply never graduate, rather than
permanently dragging the motif's centroid. How close real exposure sits to the bounded-worst-case
end vs. the cushioned end depends on how separable genuine noise is from genuine skills in actual
play, which isn't something a synthetic Gaussian test can settle — worth watching `get_stats()`'
cluster count and `n_graduated` in a live run, per the point above.

**The "same policy" claim is exactly right for the default path, with one real exception.**
`self.agent.policy.grpo_update` is bound via `types.MethodType` directly onto the live `self.policy`
object (`agent.py:857`), and `decide()`'s normal path calls `self.policy._predict()` on that same
object (`agent.py`, `decide()`) — so on the default path, GRPO's gradient and the executed action
provably share one set of weights, not a synced copy. The exception: `decide()` has a second path,
active whenever `policy_bridge.is_in_learning_mode()` is true, that instead calls `cl_head` — a
small, separate MLP (`policy_bridge.py`) that only receives a slow, partial soft-sync of the
*encoder* from the main policy (`_soft_sync_cl_head`, `tau=0.05`, every `SYNC_EVERY=200` calls) and
never receives the GRPO-updated action head directly. So: fully direct on the default path;
partial and delayed, representation-only, while a curiosity streak has learning mode active. Worth
tracing how much of an agent's lifetime that is, if precision here matters for your purposes.

## Follow-up fix: the plan itself now has a real say in what executes

`_execute_planned_action()` computed `action_arr = self.agent.decide(obs, ...)` from `obs` alone —
`action` (the plan step deliberate()/generate_plan() picked for this tick) was used only for the
trailing debug log line. The specific sequence deliberation chose never touched what actually got
sent to Minecraft; only the raw policy did.

**Considered and rejected: making the plan fully override the policy's output.** That would make
`_execute_planned_action()` behave exactly like the old template-restricted design this whole fix
exists to move away from — planner.py's own docstring is explicit that "the agent chooses freely"
and that real-time behavior shouldn't be "constrained to the template-based one-hot vectors." A
hard override re-imposes precisely that constraint at the last possible moment, just via a
different code path.

**What's implemented instead:** `CognitiveLoop.PLAN_BLEND` (default `0.25`) linearly interpolates
`action_arr` toward the plan step's own vector (via the same `_action_to_vector()` used everywhere
else, so a discovered skill's raw centroid works here too, not just hand-authored templates),
clipped to `[-1, 1]`. `0.0` recovers the old behavior; `1.0` would be the full override rejected
above. Verified in isolation: at `w=0.25` the output sits a quarter of the way from the free
policy sample toward the plan, proportionally, at every weight tested (0, 0.25, 0.6, 1.0), always
bounded. This is the same bias-not-restriction principle as the `EmergentSkillPool` fix, applied
one level down, where it changes actual per-tick behavior rather than the training signal.

This nudge applies identically whichever internal path produced `action_arr` — the main policy or
`PolicyBridge`'s `cl_head` during learning mode — since it acts on `decide()`'s returned vector, not
on how it was produced. It does not need, and isn't affected by, the `cl_head` sync caveat above.

**Also fixed while in here:** `_record_skill_attempt()` had the identical one-hot-collapse bug —
it built `action_taken` via raw `ACTION_TYPE_INDEX.get(step['type'], 0)`, so a discovered skill's
"improving" comparison against `best_action_vector` would have silently compared against whatever
template sits at index 0 instead of the skill actually attempted. Now routed through
`_action_to_vector()` like every other site.

**Not yet tuned:** `PLAN_BLEND = 0.25` is a reasonable starting point, not a derived number — watch
how plan steps actually land in a live run before trusting it. If plans feel like they're never
followed, raise it; if free exploration feels suppressed, lower it.

## Related Work (closest prior art, and why this differs)

- **Voyager** (Wang et al., 2023) — an LLM-driven Minecraft agent with an ever-growing *code* skill
  library, added on binary success/verification via GPT-4 self-checks. Closest "grow a skill
  library from experience in Minecraft" precedent, but it fits a black-box-LLM agent with no
  gradient signal. This system already has a continuous reward signal (`reward_system.py`) and a
  world model — a reward-*graded* clustering approach (this patch) uses that substrate directly
  instead of needing an LLM in the loop to write and verify code.
- **Unsupervised skill discovery** (DIAYN, DADS, METRA, and clustering-based variants like CeSD) —
  the formal name for "let behavioral categories emerge without being told what they are," usually
  via a mutual-information objective between skill and state, needing a separate discriminator
  network and no reward at all. More general than this patch, and a reasonable escalation if the
  reward-gated clustering here turns out too conservative — but a bigger lift than reusing the
  `OnlineVisualVocabulary` pattern already living in this codebase.
- **DeepMind's "Open-Ended Learning Leads to Generally Capable Agents" (XLand)** — generally
  capable behavior came from training successive *generations* of agents, each partly distilled
  from the previous generation before free reinforcement learning, rather than one lifetime
  learning from an absolute blank slate. Directly supports the "Suggested next step" above:
  instinct-carried-across-generations isn't a compromise of "learn from scratch," it's how
  open-ended learning has actually been made to work at this scale — and it maps onto
  `breeding_system.py`, which this project already has.

## Files Modified
1. **`py_backend/ai_core/emergent_templates.py`** (new) — `EmergentSkillPool`.
2. **`py_backend/ai_core/continual_learner.py`**
   - `__init__`: instantiate `self.skill_pool`.
   - `learn_from_buffer()`: feed the batch into the pool; push graduates via `planner.add_template()`.
3. **`py_backend/ai_core/brain_core.py`**
   - New module-level `_action_to_vector()` helper.
   - `deliberate()`: read `agent.planner.templates` instead of the frozen `DEFAULT_TEMPLATES`.
   - `_deliberate_with_world_model()`, `_deliberate_with_table()`, `_world_model_action_value()`:
     use `_action_to_vector()` instead of inline one-hot construction.
4. **`py_backend/ai_core/planner.py`**
   - `_run_imagination()`: use `_action_to_vector()` (imported from `brain_core.py`).
5. **`py_backend/ai_core/cognitive_loop.py`**
   - New `CognitiveLoop.PLAN_BLEND` class constant (default `0.25`).
   - `_execute_planned_action()`: nudge `action_arr` toward the plan step's vector instead of
     discarding `action` entirely.
   - `_record_skill_attempt()`: use `_action_to_vector()` instead of raw `ACTION_TYPE_INDEX` lookup.

All five touched/added files were compiled
(`python3 -m py_compile`); `EmergentSkillPool` was unit-tested standalone (including a dedicated contamination sweep, see above) and the `PLAN_BLEND` interpolation was checked numerically in isolation. The rest of the stack (Avalanche, the live world model, Minecraft) wasn't exercised end-to-end, since that needs your runtime environment.
