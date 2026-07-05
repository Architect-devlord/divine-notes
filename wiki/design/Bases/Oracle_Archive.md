---
type: base
category: god-agent
serves: "[[wiki/design/Robots/Oracle|Oracle]]"
status: ingested
---

# 🔹 Concept: Oracle Archive

💡 **Concept**

The Oracle Archive is both a base for the [[wiki/design/Robots/Oracle|Oracle]] robot itself and the long-term memory infrastructure for the entire Divine World fleet — every other base's BrainCapsule sync eventually lands here (see [[wiki/design/Robots|Robots]] → Memory Architecture). Built on the [[wiki/design/Robots|Robots]] → Shared Base Standards baseline, with power and networking treated as the base's real specialty rather than an afterthought, since this is the one facility every other base depends on indirectly.

---

## 🔹 Server Rack & NAS

- A small rack-mounted NAS (a TrueNAS- or Unraid-style build is a reasonable, well-documented default for a homelab-scale deployment like this) running redundant storage — mirrored or parity-protected, since this is the one place in the whole project where losing data means losing an agent's actual history, not just a day's uncharged battery.
- Sized generously beyond current needs: BrainCapsule backups from every active agent, plus the Oracle's own aggregated world model, accumulate faster than it's tempting to plan for early on.

## 🔹 Antenna Mast

- A mounting point for a fixed, higher-power counterpart to the Oracle robot's own staff-mounted LoRa mesh radio (see [[wiki/design/Robots/Oracle|Oracle]] → Staff Antenna System) — a stationary base-station node typically gets better range and reliability than a robot-mounted one, since it isn't power- or weight-constrained the way the mobile version is.
- Positioned for maximum line-of-sight coverage over wherever the other bases and robots typically operate, following the same "place a repeater on the high point between two spots that can't see each other" logic any LoRa mesh deployment uses.

## 🔹 UPS-Backed Power

- An uninterruptible power supply sized to keep the NAS and networking gear up through a normal power blip — not a full disaster-recovery generator, just enough runtime to survive the kind of short outage that would otherwise corrupt an in-progress BrainCapsule write.
- This matters more here than anywhere else in the fleet: per [[wiki/design/Robots|Robots]] → Compute Architecture, individual robots save their own capsules on a roughly one-minute cadence and can tolerate losing that last minute — but the Archive holds everyone's long-term history at once, where a bad write is a much bigger loss.

## 🔹 Memory Synchronization Hub

- The landing point for every other base's periodic BrainCapsule sync (see [[Common_Agent_Base]] and each God Agent base's own networking section) — receives, verifies, and archives incoming backups rather than just passively storing whatever arrives.
- Runs the memory decay process described in [[wiki/design/Robots|Robots]] → Memory Architecture: compressing low-importance routine memories over time while keeping high-importance ones (major discoveries, near-death experiences, personality-shifting events) indefinitely.

---

## 🔹 Suggested Layout

| Zone | Purpose |
|---|---|
| Rack | NAS with redundant/parity storage, sized well beyond current needs |
| Antenna mast | Fixed, higher-power LoRa mesh base station |
| Power | UPS sized for short-outage ride-through, not full disaster recovery |
| Sync hub | Receives and verifies every other base's periodic BrainCapsule sync |
| Charging dock | Standard humanoid-footprint dock, same scale as the Oracle robot itself |

---

💡 **Key Insight**

Every robot in the fleet can afford to lose a minute of unsaved experience to a crash — that's what the local one-minute save cadence is built around. The Archive is the one place in the project that can't afford the equivalent loss, because it's not holding one agent's last minute, it's holding everyone's history at once. Its design should be boring and redundant on purpose.
