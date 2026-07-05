---
type: base
category: god-agent
serves: "[[wiki/design/Robots/Elder_Guardian|Elder_Guardian]]"
status: ingested
---

# 🔹 Concept: Elder Guardian Temple

💡 **Concept**

The Elder Guardian Temple is a genuinely dual-zone base — wet and dry — because the robot it serves is genuinely dual-domain. Built on the [[wiki/design/Robots|Robots]] → Shared Base Standards baseline, with one deliberate deviation: charging here uses a contactless/inductive dock rather than the shared exposed-contact standard, since exposed charging contacts and "routinely wet robot" are a bad combination to normalize even if the dock itself stays dry.

---

## 🔹 Wet Zone: Test Tank

- A tank sized for full-submersion testing — doesn't need to be large, just deep and wide enough for the Elder Guardian to actually swim a few body-lengths and test turning, not just float in place.
- Known, marked reference distances on the tank walls/floor for validating the Ping-Sonar-class echosounder's distance readings against ground truth rather than trusting the sensor blind.
- Adjustable turbidity is a nice-to-have (something as simple as a controlled amount of fine sediment stirred in) for testing that sonar navigation genuinely still works when camera vision doesn't — the entire reason the sonar system exists in the first place.

## 🔹 Dry Zone: Land Transition Ramp

- A ramp from the tank's edge up onto dry ground, shallow enough to test the walk/swim/wade transition itself — the specific behavior the Elder Guardian's four legs are meant to handle, and one that's easy to skip testing if the base only has a tank and a charging dock.

## 🔹 Inductive Charging Dock

- A sealed, contactless charging pad rather than exposed contacts — the hull backs onto the pad, no connector needs to mate underwater or in a wet environment.
- Positioned at the tank's edge or on the dry-zone ramp, so charging naturally happens right after a swim rather than requiring a separate dry-off step first.

## 🔹 Seal & Leak Test Station

- A simple pressure or bubble-test setup for checking the hull's seal integrity after any panel has been opened for maintenance — catching a bad reseal here, on the bench, is a lot cheaper than catching it via the hull's own leak sensor mid-swim.

---

## 🔹 Suggested Layout

| Zone | Purpose |
|---|---|
| Test tank | Full-submersion swimming, sonar calibration against known distances |
| Transition ramp | Land/water gait transition testing |
| Charging dock | Inductive/contactless, at tank edge or ramp top |
| Seal test station | Post-maintenance leak verification before the next swim |
| Network | Standard wired + Wi-Fi per [[wiki/design/Robots|Robots]] → Shared Base Standards, routed to stay clear of the wet zone |

---

💡 **Key Insight**

Most of what could go wrong with this body is invisible until it's already a problem — a bad seal, a sonar reading that's drifted, a transition gait that only sort of works. The Temple's whole design is about catching those things on a bench or in a test tank, on a schedule, rather than discovering them in open water.
