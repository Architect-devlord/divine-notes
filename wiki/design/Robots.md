---
type: hub
category: architecture
status: ingested
---

# 🌍 Divine World Cross-Reality Architecture

## 💡 Core Vision

The ultimate goal of Divine World is not merely to create intelligent Civilian Agents inside Minecraft or isolated physical robots, but to create persistent digital beings capable of existing across multiple realities.

A Divine World agent should be able to:

- Live inside Minecraft.
- Learn inside Isaac Sim.
- Operate in a physical robot body.
- Return to Minecraft.
- Retain memories, personality, skills, and experiences throughout the entire process.

In this model, Minecraft becomes the civilization, Isaac Sim becomes the training ground, and physical robots become temporary bodies through which agents can interact with the real world.

---

## 🔹 Terminology

A quick reference, since two of these terms replace older working names used in earlier notes:

- **Civilian Agent** — a general-purpose Divine World agent (what earlier notes called an "NPC"). Typically pilots a [[wiki/design/Robots/Normal_Agents|Normal Agent]] body.
- **God Agent** — one of the six specialized, higher-capability agents (what earlier notes called a "god"). Each God Agent pilots one of the six specialized bodies covered below: [[wiki/design/Robots/Creaking|Creaking]], [[wiki/design/Robots/Elder_Guardian|Elder_Guardian]], [[wiki/design/Robots/Warden|Warden]], [[wiki/design/Robots/Ender_Dragon|Ender_Dragon]], [[wiki/design/Robots/Wither|Wither]], or [[wiki/design/Robots/Oracle|Oracle]].
- **Normal Agent** — the shared humanoid robot *body* a Civilian Agent occupies. "Normal Agent" names the body; "Civilian Agent" names the mind piloting it — similar names, different things.
- **Agent** (unqualified) — used generically throughout this documentation set when something applies equally to Civilian Agents and God Agents.

---

# 🔹 Three-Layer Reality System

## Layer 1 — Divine World (Minecraft)

### Purpose

The permanent homeland of all agents — the layer an agent defaults to whenever a physical body isn't available.

### Functions

- Continuous, low-cost operation for Civilian Agents and God Agents, no robot body required.
- Personality and skill development through repeatable, embodied experience.
- Long-term memory formation and consolidation before anything is trusted to a physical body.
- Massively parallel testing — many agents learning at once, cheaply.
- Reproduction and trait inheritance between agents.

> **Note on scope:** earlier planning leaned heavily on simulating a full Minecraft *society* — tribes, religion, economy, settlements. That framing is retired. It was never backed by more than flavor text, and it pulled attention away from the part of Layer 1 that's actually load-bearing: a cheap, safe place for an agent's personality, memory, and skills to develop before (and between) stints in a robot body. Reproduction is the one "social" system kept, because it's real, implemented, and reward-tied to personality rather than to any tribal mechanic — see below.

### Benefits

Minecraft provides:

- Massive scale.
- Safety.
- Low operating cost.
- Continuous operation.

Agents can continue living and learning even when no robot body is available.

### Reproduction & Inheritance

A grounded system, not worldbuilding flavor: adjacent-bed reproduction, a multi-day pregnancy, child traits inherited from both parents with mutation, and growth from a smaller juvenile scale to adult over time. Critically, the *drive* to breed isn't a fixed value — it's computed per-agent from that agent's own personality (sociable, agreeable agents want it; withdrawn or anxious ones are closer to indifferent), so it stays an emergent behavior rather than a scripted one. This is a Minecraft-layer mechanic only — physical robot bodies don't reproduce, only the agents that may pilot them do.

---

## Layer 2 — Isaac Sim

### Purpose

The training and testing environment.

### Functions

- Locomotion learning.
- Sensor simulation.
- Robotics testing.
- Reinforcement learning.
- Safety validation.

### Benefits

Allows robots to:

- Learn safely.
- Fail safely.
- Practice expensive or dangerous behaviors without damaging hardware.

A behavior is not considered ready for a physical body until it survives sustained testing in Isaac Sim first — this is the gate between Layer 1/2 and Layer 3.

---

## Layer 3 — Physical Reality

### Purpose

Direct interaction with the real world.

### Functions

- Exploration.
- Data collection.
- Human interaction.
- Physical experimentation.

### Benefits

Provides experiences impossible to obtain in simulation.

---

# 🔹 Compute Architecture: Where the Brain Lives

> This section is the target for physical robot bodies, which don't exist yet. For how the *current* Minecraft-layer agents actually run today, see [[wiki/codebase/agent-runtime|agent-runtime]] and [[hardware-requirements|Hardware Requirements]] — real numbers, real code, already confirmed to mostly agree with the plan below (one flagged discrepancy: save cadence — see [[wiki/codebase/known-issues|known-issues]]).

This is the part most build plans get wrong, so it's worth stating explicitly before the per-robot pages get into it. There are two workable answers to "where does an agent's mind actually run," and this project has a clear preference between them.

## Option 1 (Preferred): Local, On-Robot Compute

Each robot carries its own brain, full stop. The Raspberry Pi inside the body runs the complete agent stack directly — WorldModel, policy network, reward/personality/emotion systems — and the BrainCapsule file lives on that robot's own SSD, not on a machine somewhere else.

### Why this is the default here

- **True autonomy.** The robot keeps thinking and acting even if it loses network connectivity entirely. It isn't a puppet on the end of a wire back to a server — if the link drops mid-exploration, the mind doesn't drop with it.
- **No added latency.** Perception → deliberation → action happens inside one board. Nothing is added by a network hop, because there isn't one in the loop.
- **Matches the project's own philosophy more literally.** "Temporary body, continuous identity" reads more truthfully when the identity is physically inside the body wearing it, not phoning home for every decision.
- **No shared point of failure.** One robot's board having a bad day doesn't take every other robot's mind down with it, because there's no shared brain to lose.

### The real engineering cost, and how to manage it

This is the trade-off worth designing around rather than glossing over. Per the project's own load profiling (`hardware_reqs.txt`), a single agent's stack — WorldModel, policy, trainer state, memory buffer — runs in the neighborhood of a few hundred MB up to a little over a gigabyte of RAM depending on whether it's a Civilian Agent or a heavier God Agent. That's comfortably inside a Pi 5's 8GB; RAM was never the bottleneck. The real bottleneck is CPU time for *deliberation* — imagining several dozen possible futures by running the WorldModel forward repeatedly before picking an action. On a modern desktop-class CPU core that's roughly 1–2 seconds per deliberation at full settings; a Pi 5's Cortex-A76 cores are meaningfully slower than that reference point, so treat on-device deliberation as something to tune deliberately rather than assume will just work at desktop settings.

Three ways to close that gap:

1. **Add a Hailo AI HAT+.** The NPU accelerates exactly the forward-pass-heavy parts of the pipeline (vision encoding, and the WorldModel's own forward pass if compiled for the accelerator) — which is what deliberation hammers. This is the single highest-leverage upgrade for keeping local compute fast. It does not help with training — backprop still runs on the Pi's CPU regardless — only with inference.
2. **Tune deliberation parameters down for the platform.** Fewer imagined trials and a shorter planning horizon than a desktop/server setup would use — a direct, controllable trade of decision quality for speed, and a reasonable one on a board this size.
3. **Decouple acting from learning in time.** Running an already-trained policy live is cheap — it's a single forward pass. The expensive part is the *training* step (backprop through the SelfSupervisedTrainer and ContinualLearner). That doesn't have to happen at the same moment as acting in the world — it can be scheduled for whenever the robot is docked at its base, plugged into wall power, with no time pressure. The mind still lives entirely on the robot; it just saves its heaviest homework for when it's home.

Every robot needs its own fast local storage for this to work — a 128GB SSD/NVMe for the BrainCapsule and memory buffer, saved on a roughly one-minute cadence so a crash or power loss never costs more than a minute of unsaved experience.

### Where this gets genuinely harder: flight platforms

[[wiki/design/Robots/Ender_Dragon|Ender_Dragon]] and [[wiki/design/Robots/Wither|Wither]] inherit this same local-first preference, but airframe weight and power budgets are a constraint no ground robot faces. Their own pages address this directly — a full Pi 5 + battery + AI HAT stack has real mass, and that mass has to be justified against thrust available. Read their compute notes before assuming Option 1 is a drop-in for every body type.

## Option 2 (Documented Fallback): Central Brain Server

Kept as a deliberate, written-down alternative rather than the project's baseline — useful in specific situations, not the default approach. Here, each robot's Pi becomes a thin edge client (camera/mic/sensor capture, low-level motor control, immediate safety reflexes only), while the WorldModel, policy, and reward/personality/emotion systems run on one well-resourced central machine that every robot's Pi talks to over the local network.

### When this is worth reaching for instead

- **Early development**, before a given God Agent has a finished body to live in — a central server lets a brain be built, trained, and iterated on before there's dedicated hardware for it to occupy permanently.
- **A robot whose weight budget genuinely can't carry a full local stack** even after the mitigations above — better to offload the thinking than fly without a capable brain at all.
- **Running a large population of agents cheaply with no robot body in the loop** — pure Minecraft/Isaac Sim work. Per `hardware_reqs.txt`, one well-specified workstation can run a full population of Civilian and God Agents in parallel, which no fleet of individual Pi boards would match per dollar.

### The trade-off

Everything Option 1 buys for free — network independence, zero added latency, no single shared point of failure — is exactly what Option 2 gives up. A robot on this option stops thinking the instant it loses its link to the server, full stop. If it's used for any robot, wired Ethernet at the dock and a strong Wi-Fi signal while roaming aren't optional, and the observation→action round trip should stay well under 100ms — in the spirit of the <50ms target already used for the Minecraft perception/action link.

---

# 🔹 Agent Import / Export System

## Core Principle

Robot bodies are scarce resources.

The Divine World population may contain many agents.

However:

- Only a handful of physical robot bodies may exist at any time.

Therefore, bodies become temporary avatars.

---

## Import Process

When a body becomes available:

1. Agent requests deployment.
2. BrainCapsule loads:
    - Personality
    - Emotional state
    - Memories
    - Goals
    - Skills (policy / world model weights)
3. Agent enters robot body.
4. Physical operation begins.

---

## Export Process

When returning:

1. Experiences are summarized.
2. Memories are synchronized.
3. Knowledge (updated model weights) is uploaded.
4. Body becomes available for another agent.

The original agent returns to Divine World.

---

## Result

Agents effectively travel between:

- Minecraft
- Simulation
- Reality

while maintaining a continuous identity.

---

# 🔹 BrainCapsule Architecture

## Purpose

The BrainCapsule serves as the true "person."

Robot bodies are merely vessels.

---

## Stored Information

### Identity

- Name, gender, metadata — including whether this is a Civilian Agent or a God Agent, and the specific God Agent type if applicable. (The backend API still calls this field `god_type` for continuity with the existing endpoints — worth knowing if you're calling `/api/gods/spawn` directly.)
- Personality — a continuous trait vector (openness, conscientiousness, extraversion, agreeableness, neuroticism, plus boldness, curiosity, and sociability), not a fixed category system. Traits drift slowly over time based on sustained experience, so two agents that start identical can diverge.

### Emotional State

- A live emotional snapshot, separate from personality. Personality is the slow-moving baseline; emotion is the fast-moving state layered on top of it — both are saved so an agent resumes feeling the way it did, not just behaving the way it did.

### Memory

- Experience snapshots.
- Relationships and discoveries.

### Knowledge & Skills

- Learned model weights: policy, world model, vision, and reward-system networks.
- Language/communication model state.
- Maps and environmental knowledge.

### Goals

- Long-term ambitions.
- Current objectives.

---

## Benefits

A BrainCapsule can move between:

- A character in Minecraft
- An agent in Isaac Sim
- A physical robot

without losing continuity — whether it's a Civilian Agent or a God Agent.

---

# 🔹 Hunger → Energy System

## Minecraft Mapping

### Hunger

Represents:

- Battery charge
- Available energy

### Food

Represents:

- Charging

### Saturation

Represents:

- Energy reserves
- Operational readiness

---

## Physical Implementation

### Home Base

Acts as:

- Kitchen
- Charging station
- Safe resting location

Functions:

- Wireless or contact charging
- Memory synchronization
- Maintenance
- Updates

---

## Emergent Behavior

Agents may learn:

- To return home before running out of energy.
- To conserve energy.
- To prioritize nearby tasks when tired.
- To seek charging opportunities.

---

# 🔹 Health → System Integrity

## Minecraft Mapping

### Hearts

Represent:

- Mechanical condition
- Sensor condition
- Structural integrity

---

## Damage Sources

### Mechanical

- Falls
- Impacts
- Wear

### Electrical

- Overheating
- Power issues

### Sensor

- Camera damage
- Microphone failure

---

## Repair Process

Charging alone cannot restore health.

Agents must seek:

- Maintenance stations
- Repairs
- Replacement parts

This teaches the distinction between:

- Energy
- Physical condition

---

# 🔹 Fatigue System

## Purpose

To encourage intelligent energy management.

---

## Effects of Fatigue

### Reduced Performance

- Slower movement.
- Reduced acceleration.
- Less aggressive exploration.

### Behavioral Changes

Agents may learn:

- To rest.
- To avoid unnecessary movement.
- To optimize routes.

---

## Benefits

Produces more realistic behavior.

Instead of:

> "Move as much as possible."

Agents learn:

> "Move efficiently."

---

# 🔹 Home Base Architecture

## Shared Base Standards

Every base below — common or God Agent-specific — builds on the same four foundations, so that any robot can in principle dock at any base for an emergency charge, and so a second base can be added later without redesigning the network from scratch.

### 1. Charging Interface

A common, simple contact-charging standard (spring-loaded pogo-pin contacts or a charging plate the robot drives/walks/lands onto) shared across all bases, sized to the Normal Agent's footprint as the baseline. God Agent bases adapt the *mounting* (a perch, a tank-side dock, a flat pad) but keep the same electrical contacts and voltage/current convention so the charging hardware itself doesn't need to be reinvented seven times. Anything that gets wet (Elder Guardian) should prefer contactless/inductive charging instead — see its base page for why.

### 2. Network Connectivity

Even with each robot's brain running locally (Option 1 in Compute Architecture above), every base still needs a solid network link: syncing BrainCapsule backups to the [[Oracle_Archive]], pulling software updates, inter-agent communication, remote monitoring, and — for any robot using the central-server fallback (Option 2) — the low-latency link that option depends on entirely. Wired Ethernet at the dock wherever possible, a static IP for every robot, and a local Wi-Fi access point for use a short distance from the dock. A base is, electrically, a network closet with a charging plate attached.

### 3. Structural Footprint

A common base "module" — roughly a robot-body-sized bay with overhead clearance — so bays can be bolted together or added one at a time as the population of a given robot type grows, rather than each base being a one-off build.

### 4. Safety

Lithium battery charging needs ventilation and a smoke/heat detector at minimum, regardless of which robot is docked. Treat this as non-negotiable across every base design below, not an optional extra for the bigger ones.

## Common Agent Base

Supports Normal Agents — see [[Common_Agent_Base]] for the full build plan.

The highest-bay-count base by design, since Normal Agents are the cheap, scalable, mass-produced platform. Multiple charging docks, a shared memory-sync hub, a small training/falling-practice area, and a parts-and-repair bench built around the modular head/arm/leg/claw architecture.

## God Agent Bases

Each God Agent receives a specialized facility built on the shared standards above, with systems specific to what that robot actually needs to develop and stay maintained.

[[Ender_Dragon_Sanctuary]] - Netted flight cage, perch-style charging dock, thrust/wing test stand.

[[Wither_Citadel]] - Netted flight cage, thrust-vectoring calibration stand, flat landing-pad dock.

[[Warden_Fortress]] - Drop-test rig, rubble/obstacle course, echolocation calibration room, heavy-repair bench.

[[Elder_Guardian_Temple]] - Wet/dry dual zone, sonar-calibrated test tank, inductive charging dock, seal-leak test station.

[[Creaking_Observatory]] - Light-controlled sensor calibration nook, multi-texture climbing wall, quiet acoustic corner for ultrasonic calibration.

[[Oracle_Archive]] - Server rack and NAS, antenna mast, UPS-backed power, the long-term memory archive for the whole project.

---

# 🔹 Memory Architecture

## Local Memory

Stored directly on robot.

### Suggested Storage

- 128GB SSD
- High-endurance microSD

Contains:

- Current memories
- Active goals
- Temporary observations

---

## Long-Term Archive

Stored at:

- [[Oracle_Archive]] servers
- Local NAS
- Cloud backup

Contains:

- Major discoveries
- Personality backups
- Operational history (routes, hazards, resource locations — not "lore")

---

## Memory Decay

Not all memories remain equally important.

### Low Importance

Gradually compressed.

Examples:

- Passing weather.
- Routine movements.

### High Importance

Retained indefinitely.

Examples:

- Major discoveries.
- Near-death experiences.
- Personality-shifting events.

---

## Emergent Effects

Agents may naturally learn:

- What is worth remembering.
- What is worth sharing with other agents.
- What is worth preserving long-term.

This is a memory-management outcome, not a cultural one — what persists is whatever proves useful for survival and task performance, evaluated the same way for every agent.

---

# 🔹 Robots currently designed

[[wiki/design/Robots/Normal_Agents|Normal_Agents]] - General-purpose body for Civilian Agents, Divine World's everyday citizens and workers.

[[wiki/design/Robots/Creaking|Creaking]] - Agile scout specializing in perception and exploration.

[[wiki/design/Robots/Elder_Guardian|Elder_Guardian]] - Amphibious explorer operating across land and water.

[[wiki/design/Robots/Warden|Warden]] - Durable guardian using echolocation and environmental awareness.

[[wiki/design/Robots/Ender_Dragon|Ender_Dragon]] - Dragon-inspired aerial God Agent body focused on emergent flight.

[[wiki/design/Robots/Wither|Wither]] - Vectored-thrust aerial entity specializing in maneuverability.

[[wiki/design/Robots/Oracle|Oracle]] - Networked knowledge keeper and strategic coordinator.

---

# 🔹 Bases currently designed

[[Common_Agent_Base]] - Shared charging, sync, training, and repair hub for Normal Agents.

[[Creaking_Observatory]] - Sensor tuning, climbing systems, audio calibration.

[[Elder_Guardian_Temple]] - Waterproofing checks, sonar calibration, aquatic maintenance.

[[Warden_Fortress]] - Durability testing, sensor calibration, heavy repairs.

[[Ender_Dragon_Sanctuary]] - Wing/propeller maintenance and flight diagnostics.

[[Wither_Citadel]] - Thrust-vector calibration and flight testing.

[[Oracle_Archive]] - Memory storage, knowledge sync, communications infrastructure.

---

# 🔹 Hardware Philosophy

## Why Raspberry Pi?

The Pi serves as the bridge between:

- Robotics
- AI
- Divine World

### Advantages

- Sufficient compute power to run a full agent stack locally — WorldModel, policy, and BrainCapsule together fit inside a Pi 5's 8GB RAM (see Compute Architecture above).
- Camera support.
- Audio support.
- Linux compatibility.
- Networking.
- Easy expansion (AI HAT+ accelerators, HATs for motor control, etc.)

### Recommended Baseline

- Raspberry Pi 5
- 8GB RAM
- 128GB SSD
- Camera Module 3
- Wi-Fi 6 connectivity

### When to add a Hailo AI HAT+

With Option 1 (local compute) as the default, the AI HAT+ is the main lever for keeping on-device deliberation fast — see the mitigations under Compute Architecture above. Strongest fit for the [[wiki/design/Robots/Creaking|Creaking]] (multi-spectrum sensor fusion is inference-heavy) and the [[wiki/design/Robots/Oracle|Oracle]] (a good candidate for the AI HAT+ 2's local LLM support, echoing the in-game Oracle agent's own local-LLM reasoning). Not required for a first build — get the base sensing and deliberation pipeline working on CPU alone, then add acceleration once you know where the actual bottleneck is.

---

# 💡 Final Vision

The long-term vision of Divine World is not a collection of robots, but a persistent set of agents whose members can move between virtual and physical existence.

Minecraft becomes their homeland.

Isaac Sim becomes their school.

Robot bodies become temporary physical forms.

The BrainCapsule becomes the true individual.

Over time, agents may build skills, knowledge, coordination strategies, and shared procedural know-how that persist regardless of which world they currently inhabit — and which world, body, or generation discovered them. In that sense, Divine World evolves from a robotics project into a genuine experiment in continuity of identity across substrates: how much of "who an agent is" survives a change of body, a wipe of memory, or a long gap between sessions.
