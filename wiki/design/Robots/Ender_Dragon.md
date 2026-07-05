---
type: body
category: god-agent
serves: "[[Ender_Dragon_Sanctuary]]"
status: ingested
---

# 🔹 Concept: Ender Dragon Drone

💡 **Concept**

The Ender Dragon is the biomimetic aerial God Agent of the Divine World fleet — a small flapping-wing drone built to look and move like a real dragon, not just fly like one. Where the [[wiki/design/Robots/Wither|Wither]] embraces openly engineered, propeller-based thrust vectoring, the Ender Dragon commits to biological flight: a flapping-wing mechanism doing the real aerodynamic work, with a small hidden propeller handling the part flapping wings are worst at — reliable vertical lift from a standing start.

This hybrid approach is deliberate rather than a compromise. Pure flapping-wing aircraft (ornithopters) are a real, active engineering category — tailless hummingbird-scale designs exist in research labs today, capable of genuine hovering — but they're hard to get right, and takeoff/hover is usually the hardest part to nail. Pairing a flapping mechanism with a small dedicated lift propeller sidesteps the hardest part of the problem while keeping the visually and aerodynamically interesting part — forward flight, gliding, and fine maneuvering — genuinely biological.

---

## 🔹 Flight System

### Main Lift: Hidden Propeller

- A small downward-facing propeller mounted under the body, out of sight.
- Provides reliable vertical lift for takeoff — the specific maneuver flapping-wing aircraft historically struggle with most, so this offloads exactly that weak point rather than the whole flight envelope.
- Mounted on a passive or lightly-gimballed mount so it can self-adjust slightly under gravity rather than needing active stabilization of its own.

### Wings: Flapping Mechanism

- A flapping-wing mechanism in the hummingbird-drone tradition — real research platforms in this category (tailless, hummingbird- or insect-scale ornithopters) demonstrate that flapping wings can genuinely hover and maneuver, not just provide forward thrust the way a fixed wing needs airspeed to work at all.
- Provides hovering assistance, forward thrust, and agile maneuvering once the propeller has the vehicle airborne.
- Can also glide — flapping wings held rigid act as a fixed wing for a moment, cutting energy use during sustained forward flight.

### Tail: Yaw Control

- Functions as a rudder for yaw, directional stability, and fine in-flight corrections — a small, low-power addition rather than a primary control surface.

### Dynamic Balance

- The propeller handles primary lift; the wings handle fine control, agility, and glide.
- The gimballed propeller mount can self-correct slightly for tilt, damping out some of the disturbance a wing-flap introduces without needing the flight controller to actively counter every single beat.

---

## 🔹 Onboard Compute: The Real Constraint

Every other body in the fleet defaults to Option 1 (local, on-robot compute — see [[wiki/design/Robots|Robots]] → Compute Architecture) without much of a fight. The Ender Dragon is where that default actually has to be justified against physics rather than assumed.

A Raspberry Pi 5, battery, camera, and any AI HAT add real mass — mass a hand-scale flapping-wing airframe has to lift on every single wingbeat, not just carry once like a ground robot does. Worth being honest about rather than glossing over:

- **If the airframe can lift it: stay on Option 1.** A local Pi 5 running a trimmed-down deliberation setting (see the Compute Architecture mitigations — fewer imagined trials, shorter planning horizon) keeps the "temporary body, continuous identity" principle intact even on this body.
- **If payload is tighter than that:** a lighter single-board computer (something in the Pi Zero 2 W class) handling flight-critical reflexes and basic navigation locally, while heavier deliberation leans on Option 2 (the central brain server) over Wi-Fi for anything that isn't time-critical. This is a legitimate middle ground, not a failure to meet the local-first goal — it keeps the parts of the mind that *must* respond in milliseconds (stability, collision avoidance) on the robot, while the parts that can tolerate a network round-trip (longer-horizon planning) don't have to be carried in the air.
- **Either way, weigh the actual build before deciding.** This is a case where the right answer depends on real thrust and payload numbers from the specific airframe, not a rule that can be set in advance on paper — see [[Ender_Dragon_Sanctuary]] for the test stand built to produce exactly those numbers.

---

## 🔹 AI Opportunities

- Learning when to flap versus when to glide, and at what angle of attack, for a given flight goal.
- Discovering efficient ascent and descent profiles specific to this body's actual thrust and wing characteristics.
- Coordinating tail, wing, and propeller control for smooth hovering — three actuators with overlapping effects on attitude, which is exactly the kind of redundant control problem learned policies handle better than hand-tuned control loops.

---

## 🔹 Suggested Components

| System | Suggestion |
|---|---|
| Lift propeller | Small brushless motor + lightweight prop, passive/light-gimbal mount |
| Wing mechanism | Hummingbird-drone-style flapping actuator (crank/linkage driven by a small high-speed motor) |
| Tail | Lightweight servo-actuated rudder surface |
| Compute (full local) | Raspberry Pi 5, if payload allows |
| Compute (weight-constrained) | Pi Zero 2 W–class board for local reflexes, central brain server (Option 2) for deliberation |
| Sensors | Lightweight IMU as the non-negotiable minimum; camera only if payload allows |

---

## 🔹 Advantages

- Biomimetic — looks and moves like a real dragon, not an obviously mechanical drone.
- Energy efficient: the propeller absorbs the expensive part (vertical lift/takeoff), wings handle the cheap part (cruising, gliding).
- Rich control surface for the AI to explore emergent flight strategies across three actuator types.
- Small and visually unobtrusive — the hidden propeller keeps the silhouette clean.

---

💡 **Key Insight**

This hybrid design earns its complexity: a mini Ender Dragon that's practical to build, biologically inspired in the part that actually matters (forward flight and maneuvering), and gives the AI a genuinely rich set of control surfaces to learn from — while being honest that this is the one body in the fleet where "put the whole brain on board" is a real engineering question, not a given.
