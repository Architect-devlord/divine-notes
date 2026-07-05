---
type: system
status: ingested
---

# Cognitive Loop

💡 **What this is**: `ai_core/cognitive_loop.py`'s `CognitiveLoop` — the actual autonomous heartbeat of an agent. Runs every `loop_interval` seconds (default 0.5) for the lifetime of an autonomous or Minecraft-mode agent. Not used at all in chat mode — see [[agent-runtime]].

## The five-step cycle

**Perceive → Think → Reflect → Decide → Act**, then update cognitive state and sleep out the remainder of the interval.

1. **Perceive** — gathers health/hunger/emotion snapshot, recent memory events, a novelty score (inverse frequency of recent event types), an urgency score (max of emotional intensity and health deficit), and — if a `VisionAdapter` is attached — the current visual token and a vision-derived novelty contribution.
2. **Think** — runs a cheap fast-path evaluation every single cycle (`brain.evaluate_event()`), then asks the brain whether this moment is worth full deliberation (`brain.should_deliberate(novelty, urgency)`). Deliberation is genuinely expensive (see [[hardware-requirements]] for the actual cost), so it's gated, not run every tick. This is also where audio transcription, autonomous listening decisions, verbal "internal thought" generation, and web-browsing desire all get evaluated.
3. **Reflect** — turns raw thoughts into should-I flags: `should_speak`, `should_act`, `should_learn`, `should_process_file`, `should_browse_web`. File processing always wins if present; otherwise these compete on cooldowns and thresholds.
4. **Decide** — picks exactly one action type for this cycle based on the reflection flags, in priority order (file > speech > action > learn > browse).
5. **Act** — dispatches to the matching executor. Actions run concurrently with learning and continual-learning passes (`asyncio.gather`) rather than blocking on them.

## Deliberation gating — the "two-speed" design

This is the load-bearing idea in this file. `evaluate_event()` (fast path) runs every cycle with no world model involved — cheap. `deliberate()` (slow path) — which runs the WorldModel forward dozens of times to imagine possible futures — only runs when `should_deliberate()` says it's worth it. The cognitive loop controls *what* happens; the brain controls *when* deliberation is worth the cost. This split is exactly why the compute-budget discussion in the design wiki's [[wiki/design/Robots|Robots]] page matters in practice, not just in theory.

## Plan execution with mid-task interruption

When `_execute_action()` runs a multi-step plan (from `CognitivePlanner.generate_plan()`, seeded by deliberation's best action if one exists), it checks after **every single step** whether to abandon the plan:

- **Hard interrupt**: current urgency ≥ 0.75 (danger, starvation, etc.) — plan dropped immediately.
- **Soft interrupt**: a fresh deliberation finds something meaningfully better than the plan's original score (`fresh_delib.should_abort_current_plan(current_plan_score)`).

Neither check runs on the very first or very last step — only between steps, so a plan can't be interrupted before it starts or after it's already finished.

## How an agent decides what to specialize in

Two-path system, checked in priority order, feeding `active_focus_task` (which in turn gates `PolicyBridge`'s learning mode):

- **Path 2 (specialization signal, checked first)**: per-domain GRPO reward history (keyed by whatever memory-event type dominated the last 30 events — pure self-attribution, never an externally assigned label) is checked for a domain where the last 50 samples meaningfully beat the pre-existing baseline. A domain that's plateaued (near-zero variance over its last 30 samples) is excluded even if it once qualified — no point grinding a skill that's stopped improving.
- **Path 1 (fallback)**: `SkillTracker.get_weakest_task()` — practice whatever's weakest, the generalist default when nothing has emerged as a clear strength.

Entering "learning mode" itself requires a **sustained** signal, not a single spike: curiosity and surprise both above threshold for **5 consecutive cognitive cycles** (`_LEARNING_MODE_THRESHOLD_CYCLES`), specifically so one surprising tick doesn't commit the agent to practicing something on a whim.

## Self-referential skill practice

`SkillTracker` (see [[reward-and-learning-stack]]) compares what the agent actually did against what its *own most recent deliberation* said was best — there's no external teacher anywhere in this loop. Improving nudges joy/curiosity up; not improving nudges frustration up, and if frustration crosses `0.7 × persistence` (a personality trait), the agent abandons the focus task entirely rather than grinding indefinitely.

## Text training on non-chat experience

`_learning_worker()` feeds the language system from memory batches tagged `language`/`action`/`perception`, not just literal chat messages — any event without a `text` field gets serialized into a brief natural-language description first (`_event_to_text()`, e.g. `{'type': 'audio_input', 'payload': {'emotion_label': 'hostile'}}` → `"heard hostile audio"`). This is what lets the language system train on the full experience stream instead of only ever seeing social speech.
