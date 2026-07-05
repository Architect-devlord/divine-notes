---
type: body
category: god-agent
serves: "[[Wither_Citadel]]"
status: ingested
---

# 🔹 Concept: Wither Robot

💡 **Concept**

The Wither is the vectored-thrust aerial God Agent of the Divine World fleet — the engineered counterpart to the [[wiki/design/Robots/Ender_Dragon|Ender_Dragon]]'s biomimetic approach. Where the Ender Dragon commits to flapping wings, the Wither commits fully to the opposite idea: no wings, no fixed control surfaces, just three independently controllable thrust sources and an AI that has to figure out what to do with them.

This design draws on the floating, omnidirectional feel of the in-game Wither rather than its combat role. A central lift rotor carries most of the vertical load, while two servo-gimballed arm thrusters — mounted the way small hobbyist thrust-vectoring drones already do it, tilting independently to redirect thrust rather than staying fixed — give the AI direct authority over orientation and translation. There's real precedent for this exact layout: dual-gimballed-propeller drone designs already demonstrate that a counter-rotating pair of tiltable thrusters removes adverse yaw and lets the same two motors handle both horizontal thrust vectoring *and* differential-speed yaw control, without needing a fourth rotor just to cancel torque.

---

## 🔹 Body Design

### T-Shaped Chassis

- Lightweight central body module with two extended arms — the T-shape follows directly from the actuator layout (one central rotor, two arm-mounted thrusters), not styling.
- Compact enough to keep the arm-mounted thrusters' moment arm short, which matters for how quickly the vehicle can respond to a commanded attitude change.

### Internal Systems

- BrainCapsule storage, battery, flight controller, and sensor suite, all in the central module where the lift rotor's downwash and the arm thrusters' gimbal range won't be obstructed.

---

## 🔹 Flight System

### Central Lift Rotor

Handles the majority of vertical load: lift, hovering, and altitude control. Keeping one dedicated, fixed-orientation rotor for the bulk of the lift budget is what makes the two arm thrusters affordable to run at a smaller size — they're steering and translating, not straining to hold the vehicle up.

### Dynamic Arm Thrusters

Each arm carries an independent motor on a servo-controlled gimbal, giving it variable thrust magnitude and direction — up, down, forward, backward, or diagonal within mechanical limits. A counter-rotating pair (one clockwise prop, one counter-clockwise) is the standard choice for exactly this layout: it cancels the reactive torque that would otherwise twist the airframe, and lets differential thrust between the two arms provide yaw control without a dedicated yaw actuator.

### Flight Control Philosophy

No predefined movement patterns — the AI decides thrust distribution across all three rotors, learns its own stabilization, and discovers efficient flight strategies rather than following a hand-tuned control loop copied from a conventional multirotor.

---

## 🔹 Sensor Systems

- Flight sensors: IMU, gyroscope, accelerometer, barometer — the standard minimum set for any multirotor-class flight controller, learned-policy or not.
- Vision: multi-directional cameras for navigation, plus a landing-specific camera angle.

---

## 🔹 Onboard Compute: Same Constraint as the Ender Dragon

The Wither faces the identical weight-budget question raised on the [[wiki/design/Robots/Ender_Dragon|Ender_Dragon]] page, for the same underlying reason — it's a flying platform, and every gram of compute is a gram the rotors have to lift continuously, not just once.

- **Preferred:** Option 1, local compute (see [[wiki/design/Robots|Robots]] → Compute Architecture) — a full Pi 5 running the WorldModel and policy directly onboard, with deliberation settings tuned down for the platform, if the thrust-to-weight ratio supports it.
- **If not:** the same middle ground as the Ender Dragon — a lightweight board handling flight-critical stabilization and immediate reflexes locally, with Option 2 (central brain server) picking up longer-horizon deliberation over Wi-Fi. Flight stability itself should never depend on a network link staying up; only the slower "what should I do next" layer should be allowed to.
- Get real thrust and current-draw numbers from the built rotors before finalizing which of the two this ends up being — same advice as the Ender Dragon page, and for the same reason. See [[Wither_Citadel]] for the calibration stand built to produce those numbers.

---

## 🔹 Suggested Components

| System | Suggestion |
|---|---|
| Central rotor | Fixed-mount brushless motor + prop sized for the bulk of hover thrust |
| Arm thrusters | 2× brushless motor (one CW prop, one CCW prop) on 2-axis servo gimbals |
| Flight sensors | IMU, barometer — standard flight-controller sensor stack |
| Compute (full local) | Raspberry Pi 5, if payload allows |
| Compute (weight-constrained) | Lightweight flight-controller board locally + Option 2 central server for deliberation |

---

## 🔹 AI Opportunities

- Hovering, takeoff, landing, turning, and recovery, all learned rather than hand-tuned.
- Advanced maneuvers: sideways drift, stationary hover, rapid direction changes, precision positioning — the natural vocabulary of a fully vectored-thrust platform.
- Discovering thrust-distribution strategies a conventional fixed-configuration multirotor couldn't express at all.

---

## 🔹 Advantages

- Highly maneuverable — genuine omnidirectional control authority, not just pitch/roll/yaw/throttle.
- Compact, mechanically simple relative to what it can do (three thrust sources, no wings, no complex linkages).
- Rich, well-precedented control surface for emergent flight-strategy learning.

---

💡 **Key Insight**

The Wither is a flying physics problem handed to a learning system with unusually direct control authority. By giving the AI raw thrust-vectoring control rather than a pre-built flight-control abstraction, it becomes capable of discovering maneuvers a conventionally engineered multirotor was never designed to attempt.
