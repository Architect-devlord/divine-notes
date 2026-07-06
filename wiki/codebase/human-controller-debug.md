---
type: system
status: ingested
---

# Human Controller (Debug Tool)

💡 **What this is**: `human_controller_server.py` + `dw_controller.html`, a self-contained, standalone pair (own FastAPI app, own port 8888) that lets a person drive a Minecraft agent's body directly from a browser, bypassing [[brain-core]]/[[cognitive-loop]]/[[imagination-and-planning]] entirely. **Confirmed by Devlord**: its actual purpose is to verify the sensor→action pipeline end-to-end — does a keypress really turn into the right bytes, does DWClientBot really execute them, does vision/audio really come back — without needing the AI stack to be correct or even running. It is not meant as a "let a human roleplay as the AI" feature in its own right.

Not part of the Electron frontend — it doesn't import anything from `dw_agent/`, run through Vite, or share a build step. Watch for a name collision with [[electron-frontend]]'s "Controller Mode" (`ControllerSafety.jsx`): that component is a permissions dialog for the *AI* getting elevated access to a human's camera/mic/filesystem/network. This page's "controller" is the opposite direction entirely — a *human* driving an *agent's* in-game body.

## Why it has to fully take over an agent

Both pieces bind the exact same two ports a real Python AI agent would use for that character (`tcp_port` read from `agents.json`, `ws_port = tcp_port + 10_000` — matching `AgentsJsonReader.java`'s convention). The server's own docstring says to run it *instead of* the regular agent process, and warns to make sure no AI agent process is running first, since the ports will conflict. Practically, this means piloting means claiming one of the real named agents (an NPC or a specific God) from `agents.json` — there's no isolated, neutral "debug agent" identity to puppet instead.

**This is the issue Devlord flagged**: the tool's real intent is Mode 1 below, but as built it necessarily seizes a real named agent's identity and ports to do it, which plays out as "overriding an agent's controls" rather than a clean, isolated pipeline check. That override behavior is real, and could become a deliberate feature later — piloting a specific live God on purpose — just not the current intent.

**Design direction going forward (per Devlord, 2026-07-05, not yet implemented):**
- **Mode 1 — debugging / pipeline-check** (the actual current intent): verify the wire format, ports, and DWClientBot round-trip work, ideally against an isolated identity rather than a real configured agent.
- **Mode 2 — deliberate live takeover** (future, not built yet): intentionally pilot a specific already-running named agent as a real feature, rather than as a side effect of how Mode 1 happens to be wired today.

## How it actually works

`human_controller_server.py` (FastAPI, port 8888) exposes: `GET /` (serves `dw_controller.html`), `GET /api/agents` (reads `agents.json` from `~/Documents`, `~/Desktop`, or their OneDrive equivalents — first one found wins), `POST /api/connect` (picks an agent by name, starts a WebSocket listener on that agent's `ws_port` for DWClientBot to connect to, and opens a TCP connection to its `tcp_port`), and `WS /ws/human` (the browser's control channel, single-operator — one global state object, not per-session).

`dw_controller.html` is a single-file React app with no build step (React + Babel-standalone loaded from a CDN, JSX written directly in a `<script type="text/babel">` tag). WASD + pointer-lock mouse-look are sent as `type: "action"` messages on a 20Hz timer regardless of whether anything changed. Hotbar slots 1–9, and — if the selected agent is a God — a grid of ability buttons specific to that god (e.g. Warden gets `sonic_boom`/`burrow`/`emerge`; Oracle gets `summon_vexes`/`summon_fangs`, matching the god-abilities update in `log.md`'s [2026-07-04] entry) plus universal `transform`/`revert` buttons. A "Proximity Chat" tab sends/receives in-game chat as that agent. A separate "Browser" tab is just a plain `<iframe>` (defaults to minecraft.wiki) for the *human operator's* own reference while piloting — unrelated to the AI's own web-browsing permission system in [[electron-frontend]].

Actions are packed into the same big-endian binary wire format `TCPServer.java` expects (`_pack_action`): agent id, tick timestamp, four floats (move forward/strafe, yaw/pitch delta), one bit-packed flags byte (jump/sneak/attack/use/drop/open_inv/swap_hand/sprint), hotbar slot, and an optional ability name plus three float params. Frames coming back from DWClientBot are classified by `_handle_mc_frame`: JSON text is broadcast to the browser as-is; a `DWAI` 4-byte magic header selects vision/audio/JSON by a following type byte; raw JPEG magic bytes (`FF D8 FF`) are treated as vision; anything else is assumed to be raw 16-bit mono 22.05kHz PCM audio.

## Not yet confirmed

Whether the Minecraft side (`DWClientBot`/`TCPServer.java`) treats a human-controller connection any differently from a real AI-agent connection, or whether the two are genuinely indistinguishable at that layer — this page only covers the Python/browser side.
