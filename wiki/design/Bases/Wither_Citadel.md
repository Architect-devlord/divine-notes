---
type: base
category: god-agent
serves: "[[wiki/design/Robots/Wither|Wither]]"
status: ingested
---

# 🔹 Concept: Wither Citadel

💡 **Concept**

The Wither Citadel mirrors the Ender Dragon Sanctuary's basic logic — a netted flight volume plus a way to measure real thrust numbers before trusting a flight policy — but built around this body's actual mechanism: three independently thrust-vectored rotors rather than a flapping wing. Built on the [[wiki/design/Robots|Robots]] → Shared Base Standards baseline, with a flat landing-pad dock matching the Wither's likely landing gear rather than a perch.

---

## 🔹 Netted Flight Cage

- Same purpose as the Sanctuary's: a padded, enclosed volume where hovering and maneuvering policies can crash cheaply while still being tested with real airflow rather than on a static stand alone.
- Sized generously for lateral movement specifically — the Wither's whole point is omnidirectional translation via thrust vectoring, so the test volume needs room to actually demonstrate that, not just vertical hover in place.

## 🔹 Thrust-Vectoring Calibration Stand

- A fixed mount that isolates and measures each of the three rotors independently — central lift rotor, and each arm-gimbal thruster separately — so gimbal range, differential-thrust yaw authority, and per-motor thrust curves are all known quantities rather than assumptions before the AI starts learning to combine them.
- This is the equivalent of the Ender Dragon Sanctuary's test stand, adapted to three independently-controlled thrust sources instead of one propeller plus a flapping mechanism.

## 🔹 Flat Landing-Pad Dock

- A flat, marked landing pad with integrated charging contacts, sized generously since a thrust-vectored platform's landing approach can come in at an angle a fixed-wing or hover-only platform wouldn't.

---

## 🔹 Suggested Layout

| Zone | Purpose |
|---|---|
| Flight cage | Netted, padded, generous lateral room for omnidirectional testing |
| Calibration stand | Per-rotor thrust/gimbal-range measurement, all three thrusters independently |
| Landing pad | Flat, marked, integrated charging contacts |
| Network | Wi-Fi coverage through the whole cage, same reasoning as the Sanctuary |

---

💡 **Key Insight**

A vectored-thrust platform is only as good as the AI's knowledge of what each thruster can actually do — three independently gimballed, independently powered sources of thrust are a lot to learn to coordinate blind. The Citadel's calibration stand turns that into three known, measured quantities before flight training ever starts trying to combine them.
