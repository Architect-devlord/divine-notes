---
type: system
status: ingested
---

# Electron Frontend (`dw_agent/`)

💡 **What this is**: `dw_agent/electron/react-app` — a Vite + React SPA (`dw-chat-ui`) that's the human-facing window into a running agent: chat, a live "cognitive stream" of the agent's internal thoughts, a 3-D mental workspace, and a world-model activation graph. Despite living under a folder called `electron` and listing `electron` as a dependency, there's no Electron main-process file anywhere in the tree and no `electron .` launch script in `package.json` — as it stands today this runs as a plain web app served by the backend, not a packaged desktop app.

## How it finds its backend

No config file, no env var — `App.jsx` derives everything from its own URL. `packager.py` always serves this same built bundle from `backend_port + 1` for every agent, so `BACKEND_PORT = window.location.port - 1` and `BACKEND_URL`/`WS_URL` follow from that. `agentId` can't be derived the same way; it's fetched from `/status` on mount and starts as the literal string `"demo"` until that resolves.

A `VISITOR_ID` (UUID, persisted in `localStorage`, falls back to a per-session id if unavailable) is sent with every chat message as `speaker_id` — this is what lets [[reward-and-learning-stack]]'s `familiarity_r` term recognize a repeat visitor across sessions rather than treating every chat as a stranger.

## Two modes, one component tree

`mode` state (`"chat"` | `"controller"`) swaps the whole view:

- **Chat mode** — the default. Left panel: message history, a file drop zone (uploads to `/api/upload`, with an optional "Synchronous Processing" checkbox that maps to `sync: true` on the form), and a web-access manager that POSTs an enabled-domain allow-list to `/api/agents/{id}/web/allow` — the per-chat browsing permission gate. Right panel: the Mental Workspace and the Cognitive Stream (below).
- **Controller mode** — swaps to `ControllerSafety.jsx` entirely. This turned out to be big enough, and confusing enough by name, that it gets its own page: [[human-controller-debug]]. One inconsistency worth knowing about here: `ControllerSafety.jsx` hardcodes `BACKEND_URL = "http://127.0.0.1:11400"` and `agent_id: "demo"` in several places, rather than using the dynamic `BACKEND_URL`/`agentId` `App.jsx` derives — it'll silently talk to the wrong backend for any agent not on port 11400.

All chat/thought/activity data arrives over one `WebSocket` at `WS_URL` with auto-reconnect (5s backoff). Message types handled: `chat`, `agent_thought` (also flips a 2-second "Thinking" indicator), `agent_speech` (dead code path per its own comment — `cognitive_loop._broadcast_speech` actually sends `type:"chat"` with `from:"agent"`, so this case never fires in practice), `visualization_update`, `world_model_update`, and `activity_update`.

## Mental Workspace panel

Shows a text summary (object count + up to 8 labelled tags) whenever a `visualization_update` message arrives from `cognitive_loop._broadcast_mental_workspace`. A "Simulate" button opens a full-screen 3-D sandbox (`MentalMatrixSimulator.jsx`, Three.js: orbit camera, drag-to-move via a camera-facing raycast plane, add cube/sphere/cylinder primitives, import `.glb`/`.gltf`/`.obj`, basic gravity+bounce physics).

**Flagged (confirmed by Devlord, and independently by reading both sides): this simulator is not connected to the backend's Mental Matrix system at all.** `MentalMatrixSimulator.jsx` makes no fetch or WebSocket call anywhere in the file; it accepts an `onSimulationEvent` callback prop (meant to feed the modal's "Dump Buffer" export) but never actually calls it. It's a purely local, client-side physics toy — objects you add exist only in that browser tab and vanish on close. The real Mental Matrix backend (`MentalMatrixSimulation`/`MentalMatrixService`/`MentalMatrixWebSocketManager` inside `world_model.py`, see [[world-model]]) is a separate, still-not-deep-read system that this component doesn't talk to, despite the shared name and near-identical premise ("a sandbox you can watch objects move in"). [[world-model]] has been corrected to match this finding rather than its earlier "probably feeds the frontend" guess.

## World Model panel (separate system, despite the similar name)

`WorldModelVisualizer.jsx` draws a 2-D canvas node-link graph — nodes colour-coded by `type` (`vision`/`action`/`reward`/`state`), sized and pulsed by `activation` — plus a health/buffer-size readout. Driven by `world_model_update` messages, which `agent.py`'s `broadcast_world_model()` sends (that function itself wasn't traced this pass — worth confirming what it actually pulls from before assuming this shows the real `EnsembleWorldModel`'s internals). The component carries its own header comment documenting two already-fixed bugs: a hardcoded 800×600 canvas that CSS was stretching (blurry output, fixed via `ResizeObserver` keeping intrinsic size matched to layout size), and a click hit-test that used those same hardcoded dimensions instead of the actual canvas size. It also accepts an `onPredictionRequest` callback prop that, like the simulator's `onSimulationEvent`, is never actually invoked anywhere in the file.

## Cognitive Stream

A running list of `agent_thought` strings, newest at the bottom, auto-scrolling. Below it, a collapsible "System Activity Log" fed by `activity_update` messages from `agent.py`'s `broadcast_activity()`, capped at the last 20 entries, icon-coded by `activity_type` (`web`/`file`/`controller`/default).

## Sending a message

`POST /chat` with `message`, `agent_id`, `speaker_id` (the `VISITOR_ID` above), and optionally `allowed_websites` (JSON array of enabled URLs). The agent's reply normally arrives over the WebSocket first; the HTTP response is only added to the chat if the socket is disconnected, specifically to avoid double-posting the same reply.
