---
type: body
category: god-agent
serves: "[[Creaking_Observatory]]"
status: ingested
---

# 🔹 Concept: Creaking Robot

💡 **Concept**

The Creaking Robot is the perception specialist and scout of the Divine World fleet — a thin, Enderman-proportioned God Agent body built around sensing far more of the world than a human, or a standard RGB camera, ever could. Where the [[wiki/design/Robots/Warden|Warden]] trades agility for durability, the Creaking trades durability for lightness: a narrow, low-mass frame that moves quickly, balances easily, and fits into spaces a bulkier robot never could.

Its defining trait is perceptual range, not raw strength. Cameras spanning visible, infrared, and thermal bands; a microphone array that hears well past the edges of human hearing; and a body built to climb, squeeze through gaps, and recover its own footing after a fall — all in service of one job: go where it's inconvenient to send anything else, and bring back what it found.

---

## 🔹 Body Design

### Thin, Elongated Structure

- Enderman-inspired silhouette: tall, narrow, long-limbed.
- Low overall mass for agility — a lighter body needs less torque to accelerate, decelerate, and correct balance, which matters more here than raw strength ever would.
- Optimized for stealth and reconnaissance rather than payload capacity.

### Flexible Mobility

- Joints with a wide range of motion support walking, crawling, squeezing through narrow gaps, and climbing over obstacles.
- Arms double as locomotion aids, not just manipulators — see Locomotion below.

### Weight Distribution

- Batteries stored low, inside the legs rather than the torso.
- Lowers the center of gravity, which does double duty on a frame this narrow: it improves standing stability and gives the fall-recovery behavior (see Locomotion) more margin to work with.

---

## 🔹 Sensor Systems

### Multi-Spectrum Vision

- Visible-light (RGB) cameras for standard perception.
- Infrared and near-infrared cameras for night vision.
- Thermal imaging for heat-signature detection — useful for finding warm objects, active machinery, or living things in the dark or through light smoke.
- Optional ultraviolet sensing as a later add-on.
- Sensor fusion across all of the above is a learned skill, not a fixed rule: the agent decides which sensing mode to trust in a given moment (thermal over RGB in smoke, for instance) rather than following a hardcoded priority order.

### Audio Perception

- A directional microphone array, not a single mic — locating a sound's direction of arrival needs at least a few spaced elements to triangulate, the same underlying principle bat-inspired echolocation robots use for their receiver pairs (see [[wiki/design/Robots/Warden|Warden]] for the active version of this same idea).
- Noise cancellation and filtering across the full captured band.
- Frequency coverage spans the human hearing range plus ultrasonic, and optionally infrasonic and structure-borne vibration.

---

## 🔹 Sound Generation System

### Audio Output

- Human-range audio for voice communication, alerts, and direct interaction.
- Ultrasonic output for robot-to-robot communication, environmental scanning, and low-key "stealth" signaling that doesn't broadcast to anyone standing nearby.
- Infrasonic output kept as an experimental/optional channel rather than a core feature.

### Safety Indicator

- A small red LED activates any time the robot emits a frequency outside normal human hearing range, so anyone nearby has a visible cue that *something* is being transmitted even if they can't hear it.
- The emission event is logged to the BrainCapsule's memory alongside whatever triggered it, so there's a record of when and why the robot used a non-human-audible channel.

---

## 🔹 Locomotion

### Learning-Based Movement

The Creaking isn't hand-scripted to walk a specific way. Like every other Divine World body, walking, balancing, turning, and terrain adaptation are learned in Isaac Sim first (see [[wiki/design/Robots|Robots]] → Layer 2) and only trusted on hardware once validated there.

### Arm-Assisted Mobility

Arms are treated as locomotion tools, not just manipulators:

- Standing back up after a fall.
- Stabilizing during difficult terrain.
- Pulling the body over obstacles.
- Assisting climbing.

### Emergent Gait

Because gait isn't hand-scripted, the learning process is free to discover things a human designer might not think to specify: an energy-efficient cruising gait, a quieter stealth gait, a faster-but-noisier gait for open ground, and recovery behaviors specific to this body's own proportions and mass distribution.

---

## 🔹 Climbing System

### Gecko-Inspired Adhesion

Dry, gecko-inspired adhesive pads are the primary climbing mechanism, not suction — a deliberate choice. Dry adhesives (microstructured pads working on the same van der Waals principle real geckos use) draw far less power than active suction and grip a wider range of surfaces, including ones too porous or uneven for a suction cup to seal against. Climbing-robot research and commercial adhesive products have converged on the same basic pattern: a segmented or "mushroom-cap" microstructured pad on a compliant backing, engaged by shear load and released by peeling rather than pulling straight off.

- Reusable, low-power grip surfaces on hands and feet.
- Peel-to-release mechanics rather than active suck/release cycling — simpler and lower-power.

### Suction as a Secondary Option

Active suction cups stay worth keeping in reserve for very smooth, non-porous surfaces (glass, polished metal) where dry adhesive grip is weaker — a supplement to the gecko pads, not a replacement, given the power cost of running a pump continuously while climbing.

### Climbing Applications

- Walls and vertical surfaces.
- Rubble and irregular obstacles.
- Elevated observation points for a wider sensor vantage.

---

## 🔹 Onboard Compute

Runs Option 1 (local, on-robot compute — see [[wiki/design/Robots|Robots]] → Compute Architecture) like the rest of the ground-based fleet. The Creaking is the strongest candidate in the whole lineup for a Hailo AI HAT+: multi-spectrum sensor fusion (RGB + IR + thermal + audio, weighted per-moment) is exactly the inference-heavy workload the accelerator is built for, and offloading it keeps the WorldModel's own deliberation cycle from competing with sensor processing for the same CPU cores.

---

## 🔹 Suggested Components

| System | Suggestion |
|---|---|
| Compute | Raspberry Pi 5, 8GB + Hailo AI HAT+ (13 or 26 TOPS) |
| Vision | RGB camera, IR/night-vision camera, thermal camera module (e.g. FLIR Lepton-class), optional UV sensor |
| Audio in | 4+ element directional MEMS microphone array covering the ultrasonic range |
| Audio out | Human-range speaker + ultrasonic transducer, paired with the red status LED |
| Climbing | Gecko-style microstructured dry adhesive pad sheets on hands/feet; small active suction cup as a secondary option |
| Power | Battery packs distributed inside the leg modules, not the torso |

---

## 🔹 AI Opportunities

- Learning to combine RGB, IR, thermal, and audio into one coherent read of a scene, and knowing which channel to trust when they disagree.
- Discovering stealth movement patterns that reduce detectable signature — visual, acoustic, or both.
- Scouting ahead of other robots and reporting findings back, especially useful in range of the [[wiki/design/Robots/Oracle|Oracle]]'s sensor-sharing network.
- Detecting temperature shifts, unusual sounds, or motion patterns other bodies would simply miss.

---

## 🔹 Advantages

- Exceptional perception range across multiple spectra and frequency bands.
- Operates effectively in darkness, smoke, or fog.
- Detects sound beyond human hearing in both directions — listening and producing.
- Light and agile, with genuine climbing capability.
- Strong sensor-fusion learning platform.

---

💡 **Key Insight**

The Creaking is the eyes, ears, and scout of the Divine World fleet. While other God Agents specialize in strength, flight, or coordination, the Creaking specializes in *finding out* — perceiving what other bodies can't, reaching where other bodies can't fit, and reporting back what it learns. Its combination of multi-spectrum vision, wide-range audio, and low-power climbing makes it the natural first body to send anywhere unknown.
