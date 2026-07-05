---
type: body
category: god-agent
serves: "[[Oracle_Archive]]"
status: ingested
---

# 🔹 Concept: Oracle Robot

💡 **Concept**

The Oracle is the coordinator, strategist, and information nexus of the Divine World fleet — the one God Agent built around knowledge and awareness rather than a physical specialty. Where the [[wiki/design/Robots/Ender_Dragon|Ender_Dragon]] rules the skies and the [[wiki/design/Robots/Warden|Warden]] dominates harsh terrain, the Oracle's strength is scale of perspective: it doesn't just sense its own surroundings, it extends its senses through every other agent in communication range.

Modeled on the appearance and role of a Grand Warden — a tall, staff-carrying humanoid figure — the Oracle's staff is a literal communications antenna, not just a visual motif. Every active Civilian Agent or God Agent within range effectively becomes an extension of the Oracle's own sensory network, letting it build a far larger picture of the world than any single body's sensors could produce alone.

This makes the Oracle a bridge between individual and collective intelligence: every other body experiences the world directly, one set of sensors at a time; the Oracle experiences it through the whole network at once.

> **This page is about the God Agent body.** The actual codebase has a second, unrelated "Oracle" — a personal Ollama-backed tutor NPC with nothing to do with this robot. See [[wiki/codebase/oracle-two-systems|oracle-two-systems]] before touching anything Oracle-related in code — mixing the two up is an easy mistake.

---

## 🔹 Body Design

### Grand Warden-Inspired Structure

- Tall humanoid frame, larger than a [[wiki/design/Robots/Normal_Agents|Normal Agent]] body, with long limbs and a reinforced torso.
- Built more for endurance and presence than physical labor — a stable walking platform rather than a load-bearing one.
- A distinctive silhouette, deliberately: other agents should be able to identify "that's the Oracle" from a distance without checking an ID tag.

---

## 🔹 Staff Antenna System

The staff is a genuine multi-purpose antenna mast, not a prop. Framing it as a long-range mesh radio node makes this concretely buildable rather than aspirational: a LoRa-based mesh networking stack (Meshtastic is the mature open-source reference point here — off-grid, license-free ISM-band operation, self-healing multi-hop routing, no cellular or internet dependency) is a realistic, off-the-shelf way to deliver exactly the "network relay" and "position beacon" role the staff is meant to serve, and it's specifically designed for the low-bandwidth telemetry and short-message traffic this use case actually needs, rather than trying to push camera feeds over it.

Functions:

- Long-range communications and signal amplification, mounted as high as the frame allows for a genuine line-of-sight range advantage.
- Network relay — the Oracle can act as an intermediate hop for two other agents that are in range of it but not each other, the same way a LoRa mesh repeater node extends coverage between two points that can't see each other directly.
- Position beacon and emergency communication channel.

This is a longer-range, lower-bandwidth channel than the Wi-Fi link the local-compute architecture depends on for BrainCapsule sync and archive uploads (see [[wiki/design/Robots|Robots]] → Compute Architecture). The staff's job is coordination and reachability over distance, not moving large amounts of data.

---

## 🔹 Sensory Systems

### Native Vision

- Stereo RGB, wide-angle, night-vision, and infrared cameras — the Oracle's own eyes are close to a superset of what a [[wiki/design/Robots/Creaking|Creaking]] carries, since it needs to interpret sensor data *from* other bodies and benefits from firsthand experience with the same sensing modes.

### Native Audio

- Directional microphone array with noise cancellation, for conversation, environmental monitoring, and robot-to-robot audio.

---

## 🔹 Collective Perception System

### Distributed Vision & Hearing

The Oracle can pull camera feeds, sensor data, and audio events from any Divine World agent within communication range — Normal Agents, Creakings, Elder Guardians, Wardens, Ender Dragons, and Withers alike — rather than being limited to its own sensors.

### What This Actually Requires

Not magic — a defined data contract every body already speaks, since they all report through the same communication protocol back to their own BrainCapsule and, optionally, to the Oracle. The Oracle's job is aggregation and fusion across many simultaneous streams, a genuinely different computational problem from any single body's perception task, and one reason its own compute stack should be treated as the most demanding in the fleet, not the lightest.

---

## 🔹 Knowledge Network

### Collective Awareness

Combining observations across multiple locations, multiple body types, and multiple sensor modalities into one coherent situational picture — better decision-making and planning than any individual agent could manage from its own sensors alone.

### World Model

The Oracle continuously builds and maintains maps, event histories, and known resource/hazard locations — a shared operational picture other agents can query, rather than each agent maintaining an isolated, incomplete map of its own.

---

## 🔹 Memory Systems

### Long-Term Archive

The Oracle's own compute node doubles as the fleet's long-term memory archive — see [[wiki/design/Robots|Robots]] → Memory Architecture and [[Oracle_Archive]] for the base built around this role specifically. Important events, discoveries, and personality/BrainCapsule backups live here, not just the Oracle's own experiences.

---

## 🔹 Onboard Compute

Runs Option 1 (local, on-robot compute — see [[wiki/design/Robots|Robots]] → Compute Architecture), and the Oracle is arguably the strongest case in the entire fleet for it: this is the one body whose entire job is reasoning over aggregated information, so keeping that reasoning local avoids adding a network hop on top of a role that's already network-dependent for its raw data.

This is also the natural candidate for a Hailo AI HAT+ 2 (Hailo-10H, 40 TOPS, 8GB onboard RAM) specifically — its onboard-LLM support would let the Oracle run local language-model reasoning directly on the robot, echoing the same kind of local-LLM approach the in-game Oracle agent already uses for its own reasoning. Given the Oracle aggregates data from potentially several other agents at once, it's also the one body where reaching for Option 2 (central brain server) as a *permanent* choice — rather than just a fallback — is most defensible, since it's already the fleet's coordination hub and a natural place to also host shared compute if the population of active agents grows large enough to need it.

---

## 🔹 Suggested Components

| System | Suggestion |
|---|---|
| Compute | Raspberry Pi 5, 8GB + Hailo AI HAT+ 2 (local LLM support) |
| Storage | Large SSD/NVMe — the Oracle's archive role needs meaningfully more than the 128GB baseline other bodies use |
| Communications | LoRa mesh radio module (Meshtastic-compatible) on the staff mast, plus standard Wi-Fi for high-bandwidth sync |
| Vision | Stereo RGB, night-vision, and IR cameras |
| Audio | Directional microphone array |
| Mobility | Humanoid locomotion, learned in Isaac Sim like every other legged body |

---

## 🔹 AI Opportunities

- Strategic planning: resource allocation, exploration planning, and risk assessment across the whole active fleet, not just itself.
- Knowledge synthesis: spotting patterns across many observations that no single agent's data alone would reveal.
- Coordinating multi-agent tasks — assigning exploration areas, relaying findings, resolving conflicting reports from different bodies.

---

## 🔹 Optional Features

- Holographic or projected display of maps, waypoints, and warnings, if a projector unit is added to the frame later.
- Remote guidance: relaying navigation assistance or mission updates to other agents mid-task.
- Acting as a mobile network relay to temporarily extend another agent's range in the field.

---

## 🔹 Advantages

- The fleet's only genuinely collective perception system.
- Long-term memory and archive hub for the whole project, not just itself.
- Real long-range communications via a purpose-built mesh radio, not just Wi-Fi range.
- Natural coordination point for multi-agent tasks.

---

💡 **Key Insight**

The Oracle isn't the strongest, fastest, or most agile body in the fleet — it's the one whose value only exists in relation to the others. Every nearby agent becomes an extension of its senses, letting it reason at a scale no single body's own sensors could support. Where every other body specializes in *acting* on the world, the Oracle specializes in *understanding* it.
