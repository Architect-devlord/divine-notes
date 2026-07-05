---
type: body
category: god-agent
serves: "[[Elder_Guardian_Temple]]"
status: ingested
---

# 🔹 Concept: Elder Guardian Robot

💡 **Concept**

The Elder Guardian is the amphibious God Agent of the Divine World fleet — built to operate across land and water rather than choosing one. Where the [[wiki/design/Robots/Warden|Warden]] specializes in harsh terrestrial terrain and the [[wiki/design/Robots/Creaking|Creaking]] in stealth and perception, the Elder Guardian's specialty is the transition itself: walking, wading, and swimming as a single continuous behavior space rather than three separate robots stitched together.

It draws on amphibious animals — platypuses, turtles, aquatic reptiles — for the underlying idea, but not for the specific mechanism. Legs alone are an inefficient, slow way to swim, so the Elder Guardian pairs a dedicated underwater thruster and rudder-tail with four walking legs, letting the agent learn its own mix of the two rather than being locked into one gait for both mediums.

A waterproof box-hull keeps the whole design simple, modular, and easy to service — closer to an ROV chassis than a legged-robot chassis, which is deliberate: most of the hard engineering problems here (sealing, buoyancy, corrosion resistance) are already solved in the marine-robotics world, and there's no reason to solve them again from scratch.

---

## 🔹 Body Design

### Amphibious Chassis

- Waterproof box-shaped hull with a sealed internal electronics compartment — the same basic approach small ROVs use, for the same reason: it's simpler to seal one compartment well than to waterproof many small joints individually.
- Corrosion-resistant materials on anything that contacts water directly (316 stainless hardware, polycarbonate or similar for external housings — the same material choices marine-thruster manufacturers make, for the same reasons).
- Modular panels for maintenance without a full teardown.

### Environmental Adaptation

- Rated for full operation on land and fully or partially submerged.
- Withstands splashes, rain, mud, and shallow submersion without special handling.
- Geometry stable enough to support walking, wading, and swimming without redesign between them.

### Weight Distribution

- Batteries and heavy components mounted low in the hull.
- Improves swimming stability (a lower center of mass relative to buoyancy resists rolling) and reduces tipping risk on uneven land terrain — the same low-and-centered principle the [[wiki/design/Robots/Creaking|Creaking]] and [[wiki/design/Robots/Warden|Warden]] use, applied to a hull instead of legs.

---

## 🔹 Locomotion Systems

### Land Movement

Four independently controlled legs.

- More stable and easier to learn than bipedal walking — fewer degrees of freedom to coordinate, and a much larger static-stability margin at every point in the gait cycle.
- Capable of traversing rough terrain and recovering from minor slips.

### Aquatic Movement

#### Primary Propulsion

A rear-mounted underwater thruster provides the bulk of forward/reverse thrust. A brushless "flooded" thruster design — motor and stator fully exposed to water rather than sealed behind a shaft seal — is the standard approach in small ROVs specifically because it removes the seal as a failure point and lets the water itself cool and lubricate the motor. The Blue Robotics T200 is the reference product for this category: rated across a wide voltage range, drop-in mountable, and well documented, making it a sensible default rather than a custom-built thruster.

#### Tail Steering

A rudder-style tail controls yaw and adds directional stability underwater — a simple, low-power complement to the thruster rather than a second propulsion source.

#### Leg-Assisted Swimming

Whether the legs contribute meaningfully to swimming (paddling, stabilizing, or staying tucked and inert) is left for the agent to discover rather than pre-specified. This mirrors the general Divine World approach to gait: define the actuators available, not the exact motion pattern.

---

## 🔹 Sensor Systems

### Vision

- RGB cameras in waterproof housings, wide-angle where possible.
- Used for navigation, object recognition, and docking/interaction tasks — primarily an above-water and clear-water sensor, since visibility underwater is often the whole problem (see Sonar below).

### Sonar System

Cameras degrade badly in murky or dark water — this is the actual reason sonar exists as a separate system here, not just an added extra. A single-beam echosounder (the same category of sensor used for ROV altimetry and obstacle avoidance — Blue Robotics' Ping Sonar is a concrete example: 115kHz, roughly 100m range, 25° beamwidth, and an open communication protocol) gives distance-to-nearest-object readings that keep working in zero-visibility water. A wider-coverage option (forward plus side, or a full mechanical scanning sonar) is a reasonable upgrade path once the basic system is proven, giving the agent a genuine 3D read of its surroundings rather than a single ping distance.

### Environmental Sensors

- Water temperature.
- Depth/pressure.
- Leak detection inside the hull — a simple conductivity-based leak sensor is cheap insurance against a slow seal failure going unnoticed until it's a real problem.
- IMU and battery monitoring, same as every other body.

---

## 🔹 Onboard Compute

Runs Option 1 (local, on-robot compute — see [[wiki/design/Robots|Robots]] → Compute Architecture), the same default as the rest of the ground-capable fleet. The box-hull design actually makes this easier than most bodies: there's no weight-vs-thrust tradeoff to fight the way the flying God Agents face, and ample room for electronics was part of the hull's brief from the start. The one real requirement specific to this body: the compute compartment needs to be sealed to the same standard as the rest of the hull, with the Pi, storage, and any accelerator all living inside that one dry compartment rather than distributed around the body.

---

## 🔹 Suggested Components

| System | Suggestion |
|---|---|
| Compute | Raspberry Pi 5, 8GB, mounted in the sealed hull compartment |
| Propulsion | Brushless flooded underwater thruster (Blue Robotics T200 or equivalent) + ESC |
| Steering | Servo-actuated rudder-style tail |
| Sonar | Single-beam echosounder (e.g. Blue Robotics Ping Sonar) as a baseline; scanning sonar as an upgrade |
| Vision | Waterproof-housed RGB camera(s) |
| Environmental | Depth/pressure sensor, water temperature sensor, leak sensor |
| Charging | Contactless/inductive dock — see [[Elder_Guardian_Temple]] for why this body specifically avoids exposed charging contacts |

---

## 🔹 AI Opportunities

- Learning when to walk, when to swim, and when to blend both during a shoreline transition.
- Sonar interpretation and murky-water navigation as a genuinely distinct skill from camera-based navigation.
- Discovering efficient patrol routes and shoreline-following techniques.

---

## 🔹 Advantages

- Genuine amphibious operation, not just splash resistance.
- The fleet's only underwater perception capability.
- Rich, open-ended locomotion-learning problem — land, water, and the transition between.
- Simple, modular hull that's easy to service.

---

💡 **Key Insight**

The Elder Guardian bridges two environments that no other Divine World body touches. Its combination of sonar, a dedicated aquatic thruster, and four adaptable legs makes the land/water transition itself the interesting problem — not a compromise between two specialties, but a genuine third one.
