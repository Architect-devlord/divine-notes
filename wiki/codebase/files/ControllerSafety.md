---
type: file
status: ingested
---
# ControllerSafety.jsx

💡 **Role**: the entire "Controller Mode" view — a permissions dialog and activity dashboard for granting an AI agent elevated access to a human's own camera/microphone/filesystem/network, gating that access behind explicit acknowledgment checkboxes. **Not** related to [[human-controller-debug]] despite the name — that page covers a completely different system (a human piloting an agent), confirmed and corrected on [[electron-frontend]]. Defines two components in one file: the default-exported `ControllerSafety` and an internal `ControllerActivityMonitor`. **Fixed as of the 2026-07-16 pull, confirmed by diff**: both components' hardcoded `BACKEND_URL`/`"demo"` constants (documented in detail below, kept for historical context) are gone — `App.jsx` now passes `agentId` and `backendUrl` down as props, threaded through to `ControllerActivityMonitor` as well, and every API call in this file uses the real values instead of guessing. The fix comment in the source names the same consequence this page already had documented, closely enough to be worth citing directly: _"was hardcoded... silently talked to the wrong backend for any agent not on port 11400. `App.jsx` already derives the real value from its own URL; use that instead of guessing here."_

## Imports

`react` (`useState`, `useEffect`, `useRef`), `lucide-react` icons, `framer-motion` (`motion`, `AnimatePresence`) — external. [[wiki/codebase/files/MessageBubble|MessageBubble.jsx]] — internal.

## Props (`ControllerSafety`)

`onModeChange`, `messages`, `sendMessage`, `inputText`, `setInputText`, `agentId`, `backendUrl` — all supplied by [[wiki/codebase/files/App|App.jsx]]. **As of the 2026-07-16 pull, `agentId`/`backendUrl` are now included** — previously absent, which was the direct cause of the hardcoding described below (kept for historical context; the fix is described above and doesn't need repeating in full here).

## Component: `ControllerSafety`

State: `showDialog`, `agreedToTerms`, `acknowledgedRisks`, `controllerActive`, `devices` (`{cameras: [], microphones: []}`), `devicesLoading`, `selectedDevices` (`{camera: 0, microphone: 0}`), `isChatOpen`, `permissions` (`{camera, microphone, filesystem, network}`, all default `true`).

- `BACKEND_URL = backendUrl` — **as of the 2026-07-16 pull**; previously a hardcoded `"http://127.0.0.1:11400"` literal, now the real prop.
- `detectDevices()` — `GET /api/controller/detect-devices?agent_id=${agentId}` on dialog open (once — guarded by `devices.cameras.length === 0`). Maps directly onto [[dw_controller]]'s `list_cameras()`/`list_microphones()` via that route's known `__new__`-based no-agent-needed construction.
- `handleActivate()` — gated behind both checkboxes; `POST /api/controller/activate` with the real `agentId`, the enabled-permission name list, the raw `permissions` object, `selectedDevices`, and `acknowledged: true`.
- `handleDeactivate()` — `POST /api/controller/deactivate?agent_id=${agentId}`.
- Render: a collapsible chat sidebar (reusing `messages`/`sendMessage` from props, via [[wiki/codebase/files/MessageBubble|MessageBubble.jsx]]), a permission-toggle grid (checkboxes disabled once `controllerActive`), an activation modal (device selection + two required acknowledgment checkboxes + Confirm/Abort), and — only while `controllerActive` — the `ControllerActivityMonitor` below.

**The hardcoded-`"demo"` count, historical — fixed as of the 2026-07-16 pull, confirmed by diff**: `agent_id: "demo"` (or `?agent_id=demo`) previously appeared four separate times across this file — three inline in `ControllerSafety` itself (`detectDevices`, `handleActivate`, `handleDeactivate`) and a fourth passed as the literal string `agentId="demo"` into `ControllerActivityMonitor` below. None of the four read `App.jsx`'s real, correctly-resolved `agentId` state, since it wasn't threaded through as a prop at the time. **The consequence this page flagged — that Controller Mode would target the wrong agent entirely for anyone not literally named `"demo"`, not just show stale data — is exactly the reasoning the fix comment in the source now cites.**

## Component: `ControllerActivityMonitor({ agentId })`

A separate function component, defined in the same file. **As of the 2026-07-16 pull, also fixed**: previously had its own separate hardcoded `BACKEND_URL` literal (two independent hardcoded copies, not one reused) — now receives `backendUrl` as a prop from the parent `ControllerSafety`, threaded through from `App.jsx`.

- Polls `GET /api/controller/status?agent_id=${agentId}` every 2 seconds via `setInterval` (cleared on unmount).
- Renders four `ActivityIndicator`s (Camera/Microphone/File Engine/Network, active state read from `status.camera_active`/`status.microphone_active`/`status.permissions.*`) and four `StatItem`s reading `status.stats.frames_processed`/`audio_chunks_processed`/`learning_events`/`files_processed`. **Confirmed matching [[dw_controller]]'s `ControllerRuntime.get_stats()` exactly, field for field** — a clean, precise cross-stack confirmation with no discrepancy found.

### `StatItem` / `ActivityIndicator`

Two more small components in this same file, purely presentational — a labelled number and a labelled icon-with-status-dot, respectively.

## Problems (faced by traditional AI systems / LLMs)

Not AI/ML-specific — a plain state-management gap. A value (`agentId`) was correctly resolved once, in a parent component, and a sibling component that needed the same value wasn't given a way to read it — so it re-declared its own, necessarily-static substitute instead.

## Solutions

**Fixed as of the 2026-07-16 pull** — this section previously said the fix wasn't made in this file; it now is. `agentId`/`backendUrl` are threaded through as props from `App.jsx`'s render call, the same way `messages`/`sendMessage` already were, rather than each component maintaining its own hardcoded guess.

## Files Required

- [[wiki/codebase/files/MessageBubble|MessageBubble.jsx]].

## Files Used In

- [[wiki/codebase/files/App|App.jsx]] — rendered in place of the entire chat view when `mode === "controller"`.