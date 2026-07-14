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
- **Controller mode** — swaps to `ControllerSafety.jsx` entirely. **Correction**: this page previously said that component "gets its own page: [[human-controller-debug]]." That's wrong, and [[human-controller-debug]]'s own text already warns about exactly this mix-up — it covers a completely different, unrelated system (`human_controller_server.py`, a separate Flask/SocketIO server for a human to pilot an agent, not part of this frontend at all) that only shares the word "controller" by coincidence. `ControllerSafety.jsx` now has its own real per-file page: [[wiki/codebase/files/ControllerSafety|ControllerSafety.jsx]]. One inconsistency worth knowing about here, expanded on that page: `ControllerSafety.jsx` hardcodes `BACKEND_URL = "http://127.0.0.1:11400"` (declared twice, once in each of the two components this file defines) and the literal `"demo"` for `agent_id` (four separate places, not passed down from `App.jsx`'s own correctly-derived `agentId` state at all, since it isn't threaded through as a prop) — it'll silently talk to the wrong backend and query the wrong agent for anything not literally named `demo` on port `11400`.

All chat/thought/activity data arrives over one `WebSocket` at `WS_URL` with auto-reconnect (5s backoff). Message types handled: `chat`, `agent_thought` (also flips a 2-second "Thinking" indicator), `agent_speech` (dead code path per its own comment — `cognitive_loop._broadcast_speech` actually sends `type:"chat"` with `from:"agent"`, so this case never fires in practice), `visualization_update`, `world_model_update`, and `activity_update`.

## Mental Workspace panel

Shows a text summary (object count + up to 8 labelled tags) whenever a `visualization_update` message arrives from `cognitive_loop._broadcast_mental_workspace`. A "Simulate" button opens a full-screen 3-D sandbox (`MentalMatrixSimulator.jsx`, Three.js: orbit camera, drag-to-move via a camera-facing raycast plane, add cube/sphere/cylinder primitives, import `.glb`/`.gltf`/`.obj`, basic gravity+bounce physics).

**Flagged (confirmed by Devlord, and independently by reading both sides): this simulator is not connected to the backend's Mental Matrix system at all.** `MentalMatrixSimulator.jsx` makes no fetch or WebSocket call anywhere in the file; it accepts an `onSimulationEvent` callback prop (meant to feed the modal's "Dump Buffer" export) but never actually calls it. It's a purely local, client-side physics toy — objects you add exist only in that browser tab and vanish on close. The real Mental Matrix backend (`MentalMatrixSimulation`/`MentalMatrixService`/`MentalMatrixWebSocketManager` inside `world_model.py`, see [[world-model]]) is a separate, still-not-deep-read system that this component doesn't talk to, despite the shared name and near-identical premise ("a sandbox you can watch objects move in"). [[world-model]] has been corrected to match this finding rather than its earlier "probably feeds the frontend" guess.

## World Model panel (separate system, despite the similar name)

`WorldModelVisualizer.jsx` draws a 2-D canvas node-link graph — nodes colour-coded by `type` (`vision`/`action`/`reward`/`state`), sized and pulsed by `activation` — plus a health/buffer-size readout. Driven by `world_model_update` messages, which `agent.py`'s `broadcast_world_model()` builds. **That function's own call sites are now confirmed, not just untraced**: a repo-wide search finds zero — `broadcast_world_model()` is defined and never called anywhere. This is a stronger finding than "shows stale data": `App.jsx` only mounts `WorldModelVisualizer` at all when `worldModelData` is truthy, and that state is only ever set inside the WebSocket's `world_model_update` handler — so the entire panel never appears in the running UI, not just renders without live updates. The component carries its own header comment documenting two already-fixed bugs: a hardcoded 800×600 canvas that CSS was stretching (blurry output, fixed via `ResizeObserver` keeping intrinsic size matched to layout size), and a click hit-test that used those same hardcoded dimensions instead of the actual canvas size. It also accepts an `onPredictionRequest` callback prop that, like the simulator's `onSimulationEvent`, is never actually invoked anywhere in the file — and two smaller things found on this pass: a `showPredictions` toggle whose state is watched by the redraw effect but never actually changes what gets drawn, and a `hoveredNode` state with real render logic behind it (a highlight outline) that nothing ever sets, since only clicks — not hovers — update selection state. Full detail: [[wiki/codebase/files/WorldModelVisualizer|WorldModelVisualizer.jsx]].

## Cognitive Stream

A running list of `agent_thought` strings, newest at the bottom, auto-scrolling. Below it, a collapsible "System Activity Log" meant to be fed by `activity_update` messages from `agent.py`'s `broadcast_activity()` — **confirmed this pass, same as `broadcast_world_model()` above: zero call sites anywhere in the repository.** The backend-driven channel (`web`/`controller`-type activities, per the icon-selection logic) never fires in the current system. The log isn't completely inert, though — `App.jsx`'s own file-upload handler (`onFileSend`) pushes a `file`-type entry into the same local `activities` state directly, client-side, with no backend round-trip at all. So in practice this log can only ever show file uploads, despite clearly being built to show three other activity types too.

## Sending a message

`POST /chat` with `message`, `agent_id`, `speaker_id` (the `VISITOR_ID` above), and optionally `allowed_websites` (JSON array of enabled URLs). The agent's reply normally arrives over the WebSocket first; the HTTP response is only added to the chat if the socket is disconnected, specifically to avoid double-posting the same reply.