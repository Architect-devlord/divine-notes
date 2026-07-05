---
type: reference
status: ingested
---

# Hardware Requirements

💡 **What this is**: the actual load profile and machine-sizing math for running the Minecraft-layer population, sourced from `hardware_reqs.txt`. This is about the server that runs Civilian/God Agents inside Minecraft — a different question from the physical-robot compute budget the design wiki's [[wiki/design/Robots|Robots]] page works through (though the two share real numbers — see below).

## Per-agent load (one Python subprocess each)

| Component | RAM | CPU |
|---|---|---|
| Python interpreter + FastAPI/uvicorn | ~150MB | baseline |
| WorldModel (6-layer multimodal Transformer) | ~300MB | heavy — deliberation |
| TransformerPolicy (SB3) | ~80MB | medium |
| SelfSupervisedTrainer optimizer state | ~200MB | heavy — backprop |
| ContinualLearner (Avalanche) + experience buffer | ~60MB | medium |
| PolicyBridge + cl_head | ~10MB | light |
| SkillTracker + misc | ~20MB | light |
| **Total per normal agent** | **~820MB** | **significant** |

God agents: same architecture + `GodAbilityHead` + heavier deliberation params, ~1.4GB RAM, ~1.5× the CPU load of a normal agent.

**The critical CPU spike is deliberation**: at `horizon=12, n_trials=40`, that's 480 WorldModel forward passes per deliberation event — roughly 1–2 seconds on a modern CPU core for a ~10M-parameter Transformer. Deliberation fires roughly every 5–10 cognitive cycles (so every 5–10 seconds per agent). With many agents staggered, that's near-constant load across cores. See [[cognitive-loop]] and [[reward-and-learning-stack]] for what deliberation actually computes.

## Full-population example (25 agents: 20 normal + 4 god + oracle-on-Pi)

```
20 normal agents  × 820MB   =  16.4GB
4 god agents      × 1.4GB   =   5.6GB
Minecraft server             =  12.0GB   (Paper, 25 agent-players + plugin, 4+ cores)
ScyllaDB                     =   8.0GB   (NVMe mandatory — see below)
OS + monitoring + buffers    =   6.0GB
──────────────────────────────────────
Minimum total                =  48.0GB
Comfortable headroom (+33%)  =  64.0GB
Growth room (future agents)  =  96–128GB
```

RAM is the hard constraint; everything else scales from it. CPU: 25 staggered agent processes alone need 20–28 cores under load; comfortable is 48–64 cores total once Minecraft + ScyllaDB + OS are added.

**Why ScyllaDB needs NVMe specifically**: 25 agents writing memory events continuously (see [[memory-and-braincapsule]]) — latency degrades badly on spinning disk at that write volume.

## Three hardware tiers

- **🟡 Minimum Viable (~$2,500–3,500)** — Ryzen 9 7950X (16c/32t), 128GB DDR5, 2TB NVMe, no GPU. Runs it, but 16 cores for 25 processes is tight; deliberation cycles queue, agent response latency can stretch to 3–5s under full load. Fine for development, not for continuous operation.
- **🟢 Recommended (~$4,500–6,500)** — Threadripper 7960X (24c/48t), 128GB DDR5 **ECC**, TRX50, 2TB NVMe + 4TB HDD, no GPU initially. 48 threads comfortably covers all three workloads simultaneously. ECC matters concretely here: agents save brain capsules every ~5 minutes (see [[memory-and-braincapsule]]) — an uncorrected memory error corrupting a capsule mid-write silently breaks an agent.
- **🔵 Future-Proof (~$8,000–12,000)** — Threadripper PRO 7965WX/7985WX, 256GB ECC (8-channel), RAID-1 NVMe specifically for brain-capsule safety, RTX 4090 reserved for a future model-serving layer.

## Why not GPU today

Current architecture is one process per agent with no shared GPU state. Adding a GPU now means 25 processes fighting over CUDA context switches — often *slower* than CPU for small Transformers at batch size 1. GPU becomes genuinely valuable once a shared model-serving layer exists — one process owning the GPU, batching inference requests from every agent. At that point a single RTX 4090 could serve all 25 agents' WorldModel inference in batches of 25, cutting deliberation from ~1.5s to ~50ms per agent. Real architectural work, not yet built.

**Can use GPU for minecraft server so that server can run smoothly even with a lot of agents**

## Pi ↔ Oracle connection note

Directly relevant to [[wiki/design/Robots|Robots]]'s physical-robot Compute Architecture discussion: the Oracle's observation payload is tiny — 128 floats × 4 bytes = 512 bytes per request. Even at 10 deliberation requests/second, that's 5KB/s. A standard LAN with <5ms RTT is more than enough. The only real requirement: give the Pi a **static IP** so the agent's HTTP client doesn't hit a DNS lookup on every single call — the same specific recommendation the design wiki's [[wiki/design/Robots|Robots]] page makes independently, confirmed here from the source.
