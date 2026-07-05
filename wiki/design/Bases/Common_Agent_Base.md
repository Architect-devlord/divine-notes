---
type: base
category: civilian-agent
serves: "[[wiki/design/Robots/Normal_Agents|Normal_Agents]]"
status: ingested
---

# 🔹 Concept: Common Agent Base

💡 **Concept**

The Common Agent Base is the shared home for every [[wiki/design/Robots/Normal_Agents|Normal Agent]] body — and, by extension, every Civilian Agent that occupies one. Because Normal Agents are the cheap, mass-produced platform (see [[wiki/design/Robots|Robots]] → Hardware Philosophy), this base is designed around *population*, not any single robot: multiple charging docks, a shared sync hub, and a repair bench built around the modular head/arm/leg/claw architecture described on the Normal Agent page.

This base implements every standard from [[wiki/design/Robots|Robots]] → Home Base Architecture → Shared Base Standards (charging interface, network connectivity, structural footprint, safety) at their baseline spec — every other base in the fleet is a variation on this one, not the other way around.

---

## 🔹 Charging Bay Row

- Multiple charging docks in a row, each using the shared contact-charging standard (pogo-pin contacts or a drive-onto plate) at the baseline spec sized for a Normal Agent's footprint.
- Docks numbered/addressed individually so the backend can track which specific robot is charging where — useful once population grows past a handful of units.
- Sized for simultaneous charging of several units at once, since this base's whole premise is scale.

## 🔹 Memory Sync & Knowledge Hub

- Wired network drop at every charging bay (see [[wiki/design/Robots|Robots]] → Compute Architecture, Option 1 default: this is for BrainCapsule backup sync and software updates, not live brain-server traffic, since Normal Agent bodies run their own compute locally).
- A local sync point that periodically pushes BrainCapsule backups toward the [[Oracle_Archive]] for long-term storage, so a base-level hardware failure never means losing more than the most recent sync interval's worth of experience.

## 🔹 Training / Practice Area

- An open floor area with a few genuinely uneven surfaces (a ramp, some loose gravel or sand patches, a step or two) — the same logic as the Warden's durability testing, scaled down: cheap, safe, repeatable ground truth for validating that a walking policy trained in Isaac Sim actually transfers to hardware before trusting it further afield.
- Enough clear floor space for basic fall-recovery practice without obstacles in the way.

## 🔹 Repair & Modification Bench

Built directly around the Normal Agent's modular architecture (see [[wiki/design/Robots/Normal_Agents|Normal_Agents]] → Modular Body Architecture):

- A bench-mounted quick-connect jig matching the same head/arm/leg/torso interface the robots use, so a damaged module can be pulled and swapped without improvising fixturing each time.
- Spare parts storage organized by module type, since damaged modules get swapped rather than repaired in place in most cases.
- A small stock of end effectors (claws, hooks, platforms) for the manipulation system's quick-connect mount, so a unit can be re-tasked without waiting on a specific part order.

---

## 🔹 Suggested Layout

| Zone | Purpose |
|---|---|
| Charging row | 4–8+ docks, expandable in additional rows as population grows |
| Network closet | Router/switch, wired drops to every dock, sync server toward [[Oracle_Archive]] |
| Training floor | Open area + a ramp, loose-surface patch, small step |
| Repair bench | Module jig, spare-parts bins by module type, end-effector stock |
| Power | Sized for simultaneous multi-unit charging — check total draw against the charging row's full dock count, not just a single unit |

---

## 🔹 Safety

Standard lithium charging precautions apply per bay, scaled up: ventilation and smoke/heat detection across the whole charging row, not just spot-checked at one dock (see [[wiki/design/Robots|Robots]] → Shared Base Standards).

---

💡 **Key Insight**

Every other base in this project is a specialized variation on this one. If the charging standard, network pattern, and safety baseline established here work cleanly at scale, every God Agent base inherits a design that's already been proven — they're just adding what's specific to their own body on top of it.
