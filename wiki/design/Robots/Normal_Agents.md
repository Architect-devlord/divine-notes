---
type: body
category: civilian-agent
status: ingested
---

# 🔹 Concept: Normal Agent Robot

💡 **Concept**

The Normal Agent Robot is the general-purpose, mass-producible body of the Divine World ecosystem — the physical form a **Civilian Agent** occupies when it steps out of Minecraft and into the real world. Where each God Agent gets a specialized chassis built around one signature capability, the Normal Agent is deliberately generic: a human-scale biped with two arms, swappable end effectors, and a modular skeleton, built to be cheap enough to produce many of, and repairable enough to keep running without specialist parts.

Its proportions follow the default Minecraft player skeleton (Steve/Alex) — not for novelty, but because a human-scale biped is the best-understood locomotion problem in robotics, has the largest body of Isaac Sim training reference to draw on, and moves through human-built spaces (doorways, stairs, tool handles) without any special-casing.

See [[wiki/design/Robots|Robots]] → Terminology for how "Normal Agent" (the body) relates to "Civilian Agent" (the mind that pilots it).

---

## 🔹 Core Computing

- Raspberry Pi 5 as the onboard controller, running the full agent stack locally: vision processing, audio processing, decision-making, and BrainCapsule storage all live on the robot itself. See [[wiki/design/Robots|Robots]] → Compute Architecture, Option 1, for why this is the default and how deliberation cost is managed on a Pi-class board.
- The Normal Agent has none of the weight or power crisis the flying God Agents face — a full Pi 5, battery, and optional Hailo AI HAT+ fit comfortably in a human-scale torso cavity, so there's little reason to reach for the central-server fallback (Option 2) here except during early development.
- Modular compute mounting so the board can be swapped for a more powerful successor later without redesigning the body around it.

---

## 🔹 Vision System

- Dual cameras mounted in the head, angled slightly outward for a wider combined field of view and basic stereo depth from parallax.
- Supports:
    - Navigation and obstacle detection.
    - Human interaction (face and gesture awareness).
    - Environmental awareness feeding the WorldModel.
- Visual features and tokens are learned by the agent's own vision pipeline rather than assigned from a fixed pretrained label set — the agent builds its own visual vocabulary from what it actually sees.
- Isaac Sim is the training ground for turning raw camera input into useful policy input before anything reaches a physical head.

---

## 🔹 Audio System

- Microphone array integrated into the head.
- Enables:
    - Voice recognition and conversation.
    - Directional sound localization.
    - Agent-to-agent communication over the robot's own speaker/mic loop, distinct from the network channel used for data sync.

---

## 🔹 Manipulation System

- Two articulated arms, each ending in a standardized quick-connect mount rather than a fixed hand.
- Modular claw-style end effectors — a Lego-compatible mounting pattern is a genuinely buildable starting point: cheap, widely available, and easy to prototype new attachments against. Support:
    - Grabbing and carrying objects.
    - Simple construction and assembly tasks.
    - Object sorting.
- End effectors swap per task — a gripper for one job, a hook or flat platform for another — without touching the arm itself.

---

## 🔹 Locomotion

- Two-legged humanoid structure, proportioned after Steve/Alex.
- Supports walking, turning, and general terrain traversal at human scale.
- Like every other Divine World body, gait is *learned*, not hand-scripted: policies train in Isaac Sim first, get validated there, and only then get trusted on physical hardware — see [[wiki/design/Robots|Robots]] → Layer 2 for why that gate exists.

---

## 🔹 Grip Enhancement

- Gecko-inspired dry adhesive pads on hands and feet, for extra surface grip, light climbing assistance, and slip recovery.
- [[wiki/design/Robots/Creaking|Creaking]] is the platform built around climbing as a specialty and covers the adhesion mechanism itself in real depth — the Normal Agent uses the same category of pad as a supplementary grip aid, not a primary capability.

---

## 🔹 Power System

- Internal battery pack positioned near the torso's center of mass, for balanced walking.
- Front-accessible battery bay for fast swaps without a full teardown.
- Sized for a full working day between charges as a starting target, refined once real power-draw data exists from a built unit.

---

## 🔹 Modular Body Architecture

- Head, arms, legs, and torso are separable modules on standardized mechanical and electrical interfaces.
- Benefits:
    - A damaged limb is a module swap, not a full rebuild.
    - New sensor or actuator variants can be trialed on one module without touching the rest of the robot.
    - Multiple Normal Agent bodies can share spare parts.

---

## 🔹 BrainCapsule Integration

Each Normal Agent body is a vessel for a Civilian Agent's BrainCapsule — see [[wiki/design/Robots|Robots]] → BrainCapsule Architecture for the full contents (personality, emotional state, memory, learned model weights, goals). The capsule is what makes the agent portable between Minecraft, Isaac Sim, and this body; the body itself holds no identity of its own.

---

## 🔹 Interaction & Coordination

- Conversational interaction with humans and with other agents — Civilian or God — over the same communication pipeline used in Minecraft.
- Task coordination with other Normal Agents or God Agents on shared jobs, including relaying an observation to the [[wiki/design/Robots/Oracle|Oracle]] rather than acting alone on incomplete information.
- Shared knowledge access through the Oracle's collective-perception network (see [[wiki/design/Robots/Oracle|Oracle]]) whenever one is in range.

---

## 🔹 Suggested Components

| System | Suggestion |
|---|---|
| Compute | Raspberry Pi 5, 8GB, optional Hailo AI HAT+ |
| Storage | 128GB NVMe/SSD |
| Vision | 2× Camera Module 3 (or equivalent), stereo-mounted |
| Audio | Small USB or I2S microphone array, 3–4 element |
| Grip | Commercial gecko-style dry adhesive pad sheet, cut to hand/foot size |
| End effectors | Servo-driven claw, Lego-compatible quick-connect mount |
| Battery | Swappable Li-ion pack, torso-mounted |
| Actuation | Standard hobby/robotics servos or small BLDC + gearbox per joint, sized against Isaac-Sim-validated torque requirements before final selection |

---

## 🔹 Advantages

- **Low cost**: Raspberry Pi and modular components keep unit cost manageable.
- **Highly repairable**: broken modules replace independently.
- **Expandable**: sensors, tools, and compute upgrade over time without a redesign.
- **Human-friendly form**: moves through human spaces and interacts naturally with people.
- **Scalable production**: identical designs across many units, sharing spare parts.

---

💡 **Key Insight**

The Normal Agent Robot is essentially the physical embodiment of a Civilian Agent: a low-cost, human-scale, modular platform whose personality, memory, skills, and learned behaviors move seamlessly between Minecraft, Isaac Sim, and the real world through the BrainCapsule system. Where the six God Agent bodies each specialize hard around one capability, the Normal Agent stays deliberately general — the workhorse that makes population scale possible.
