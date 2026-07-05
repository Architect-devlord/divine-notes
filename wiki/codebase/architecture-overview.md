---
type: overview
status: ingested
---

# Architecture Overview

💡 **What this is**: how the four physical pieces of Divine World actually talk to each other. Read this before any of the other pages — everything else assumes this map.

## The four pieces

| Piece | Language | Role |
|---|---|---|
| `DivineWorld` (server mod) | Java / Forge | Runs inside the Minecraft server. Owns world events, breeding, god spawning/disguising, the Oracle NPC tutor. |
| `DWClientBot` (client mod) | Java / Forge | Runs inside each agent's own Minecraft client instance. Captures the actual camera/audio feed an agent "sees," renders god entities, relays to the Python backend over TCP/WebSocket. |
| `py_backend` | Python / FastAPI | Where the actual thinking happens. One subprocess per agent (`ai_core/agent.py`), each its own tiny web server. |
| `dw_agent` (Electron app) | React / Electron | Human-facing control panel — agent status, the Mental Matrix simulator, a world-model visualizer, file drop zone for teaching agents. |

Isaac Sim sits alongside this as the training ground (`py_backend/isaac_sim_integration.py`, not yet deep-ingested into this wiki) — the layer where locomotion policies get trained/validated before a body ever has to try them for real, mirroring the same role it plays in the design wiki's Layer 2.

## Why one subprocess per agent

Every `NPCAgent` (see [[agent-runtime]]) is its own FastAPI process with its own port pair, not a thread inside one shared server. This is a real architectural choice, not an accident: it means one agent crashing doesn't take the others down, and it's what makes the hardware-sizing math in [[hardware-requirements]] a simple "cost per agent × agent count" calculation rather than something more entangled.

## Port scheme

Every agent gets two ports, resolved from `agents.json` by display name (starting at TCP 11401):

- **TCP action-server port** (`tcp_port`, e.g. 11401) — the Java mod's `TCPServer` connects here as a client. This is the fallback transport.
- **WebSocket backend port** (`backend_port` = `tcp_port + 10000`, e.g. 21401) — the Java mod connects here as a WebSocket client at `ws://127.0.0.1:<port>/ws/agent`. This is the primary, low-latency transport.

Both are resolved once at agent startup and persisted in the BrainCapsule (see [[memory-and-braincapsule]]) so a restarted agent reuses the same ports even if `agents.json` is briefly unreachable.

## The perception/action bridge

See [[communication-protocol]] for the full detail. Short version: `DWClientBot` captures what an agent's Minecraft client actually sees/hears and streams it over WebSocket (binary JPEG frames + MessagePack for structured data) to that agent's Python process, targeting <50ms round-trip. If the WebSocket connection drops, `run_tcp_action_loop()` (`ai_core/communication_protocol.py`) picks up as a 20Hz polling fallback — documented as a fallback, not the primary path.

`PythonBackendClient.java` is the other direction: plain HTTP from the Java server mod to the Python backend's REST API (`/api/agents/spawn_single`, the genesis endpoint, etc.) for things that aren't part of the tight perception/action loop — spawning, coordination, one-off requests.

## Packaging and deployment

Each agent, once created, gets packaged (`packager.py`) into a standalone portable bundle containing its own Minecraft account, a preconfigured Forge instance with `DivineWorld` + `DWClientBot` installed, and a bundled `UltimMC` launcher (a Minecraft instance launcher, not related to the DW project itself) — see [[known-issues]] for the packaging bugs this went through and how they were fixed. The intent is that an agent is a self-contained thing you can hand off or run independently, not something that only works inside one dev environment.

## Not yet deep-ingested here

The full Minecraft mod layer beyond what's covered on [[oracle-two-systems]] and [[known-issues]] — entity rendering (GeckoLib-based models for the Creaking, notably — same "geo" naming convention as the design wiki's gecko-adhesion theme, whether that's intentional or a happy coincidence isn't confirmed), the individual per-god client AI classes (`AICreaking.java`, `AIWarden.java`, etc. in `DWClientBot`), the event handlers (`BreedingEventHandler`, `GodSpawnHandler`, `GodDisguiseHandler`, `ProximityChatHandler`), and the `dw_agent` Electron frontend (`WorldModelVisualizer.jsx`, `MentalMatrixSimulator.jsx`) all exist and are referenced by name in code I've read, but haven't been read in enough depth for their own page yet. Good candidates for the next ingest pass.
