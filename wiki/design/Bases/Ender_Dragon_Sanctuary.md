---
type: base
category: god-agent
serves: "[[wiki/design/Robots/Ender_Dragon|Ender_Dragon]]"
status: ingested
---

# 🔹 Concept: Ender Dragon Sanctuary

💡 **Concept**

The Ender Dragon Sanctuary is built around the two things a small flapping-wing platform actually needs validated before it's trusted to fly unsupervised: the flight mechanism itself, and a safe volume to fail in while that mechanism is still being tuned. Built on the [[wiki/design/Robots|Robots]] → Shared Base Standards baseline, with a docking mount instead of a floor-level charging plate — this body lands on a perch, not a drive-onto pad.

---

## 🔹 Netted Flight Cage

- An enclosed netted volume sized for genuine hovering and short forward-flight testing, not just a static test stand — the point is to validate flight behavior with real airflow and real disturbance, in a space where a crash costs nothing but a reset.
- Padded floor and walls, since early-stage flight policies (especially anything learned rather than hand-tuned) will crash during training, reliably and often, and that needs to be cheap rather than damaging.

## 🔹 Perch-Style Charging Dock

- A raised perch the Ender Dragon lands on directly, with charging contacts built into the landing point — avoids needing a separate "land, then walk to a charger" step for a body that doesn't really walk.
- Positioned at the edge of the netted cage so charging and flight testing share the same space rather than requiring a second area.

## 🔹 Thrust & Wing Test Stand

- A fixed mount that holds the airframe stationary while measuring actual thrust output from the lift propeller and actual force/frequency from the flapping mechanism — exactly the real numbers referenced on the [[wiki/design/Robots/Ender_Dragon|Ender_Dragon]] page's Onboard Compute section as the deciding factor between a full local Pi 5 and the lighter weight-constrained compute option.
- This is where "can this airframe actually carry a Pi 5 or not" gets answered with data instead of guessed at.

---

## 🔹 Suggested Layout

| Zone | Purpose |
|---|---|
| Flight cage | Netted, padded, sized for real hover/forward-flight testing |
| Perch dock | Landing-integrated charging contacts |
| Test stand | Static thrust/wingbeat measurement, feeds the compute-budget decision |
| Network | Wi-Fi coverage through the whole cage — critical if this body ends up on the Option 2 compute path |

---

💡 **Key Insight**

Everything about a small flapping-wing platform is a numbers game — thrust, mass, and power draw all have to actually add up before any flight policy is worth trusting. The Sanctuary's test stand exists to produce those numbers directly, rather than inferring them indirectly from how a flight attempt happened to go.
