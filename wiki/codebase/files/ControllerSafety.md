---
type: file
status: ingested
---
# ControllerSafety.jsx

💡 **Role**: the entire "Controller Mode" view — a permissions dialog and activity dashboard for granting an AI agent elevated access to a human's own camera/microphone/filesystem/network, gating that access behind explicit acknowledgment checkboxes. **Not** related to [[human-controller-debug]] despite the name — that page covers a completely different system (a human piloting an agent), confirmed and corrected on [[electron-frontend]]. Defines two components in one file: the default-exported `ControllerSafety` and an internal `ControllerActivityMonitor`, each with their own separate hardcoded backend constants (see below).

## Imports

`react` (`useState`, `useEffect`, `useRef`), `lucide-react` icons, `framer-motion` (`motion`, `AnimatePresence`) — external. [[wiki/codebase/files/MessageBubble|MessageBubble.jsx]] — internal.

## Props (`ControllerSafety`)

`onModeChange`, `messages`, `sendMessage`, `inputText`, `setInputText` — all supplied by [[wiki/codebase/files/App|App.jsx]], genuinely sharing its chat state rather than maintaining a separate one. **Notably absent: no `agentId`, `BACKEND_URL`, or `WS_URL` prop of any kind** — despite `App.jsx` having correctly derived all three of these itself. This is the direct cause of the hardcoding below, not an independent choice made inside this file.

## Component: `ControllerSafety`

State: `showDialog`, `agreedToTerms`, `acknowledgedRisks`, `controllerActive`, `devices` (`{cameras: [], microphones: []}`), `devicesLoading`, `selectedDevices` (`{camera: 0, microphone: 0}`), `isChatOpen`, `permissions` (`{camera, microphone, filesystem, network}`, all default `true`).

- `BACKEND_URL = "http://127.0.0.1:11400"` — hardcoded, declared at the top of this component.
- `detectDevices()` — `GET /api/controller/detect-devices?agent_id=demo` on dialog open (once — guarded by `devices.cameras.length === 0`). Maps directly onto [[dw_controller]]'s `list_cameras()`/`list_microphones()` via that route's known `__new__`-based no-agent-needed construction.
- `handleActivate()` — gated behind both checkboxes; `POST /api/controller/activate` with `agent_id: "demo"`, the enabled-permission name list, the raw `permissions` object, `selectedDevices`, and `acknowledged: true`.
- `handleDeactivate()` — `POST /api/controller/deactivate?agent_id=demo`.
- Render: a collapsible chat sidebar (reusing `messages`/`sendMessage` from props, via [[wiki/codebase/files/MessageBubble|MessageBubble.jsx]]), a permission-toggle grid (checkboxes disabled once `controllerActive`), an activation modal (device selection + two required acknowledgment checkboxes + Confirm/Abort), and — only while `controllerActive` — the `ControllerActivityMonitor` below.

**The hardcoded-`"demo"` count, precise**: `agent_id: "demo"` (or `?agent_id=demo`) appears **four** separate times across this file — three inline in `ControllerSafety` itself (`detectDevices`, `handleActivate`, `handleDeactivate`) and a fourth passed as the literal string `agentId="demo"` into `ControllerActivityMonitor` below. None of the four ever reads `App.jsx`'s real, correctly-resolved `agentId` state, since it isn't threaded through as a prop. **Practical consequence, not just a style note**: for any live agent whose real ID isn't literally `"demo"`, every Controller Mode API call targets the wrong agent entirely — not stale data, a wrong target.

## Component: `ControllerActivityMonitor({ agentId })`

A separate function component, defined in the same file, with its **own separate** `BACKEND_URL = "http://127.0.0.1:11400"` declaration — not shared with the parent's copy above; two independent hardcoded constants, not one reused twice.

- Polls `GET /api/controller/status?agent_id=${agentId}` every 2 seconds via `setInterval` (cleared on unmount).
- Renders four `ActivityIndicator`s (Camera/Microphone/File Engine/Network, active state read from `status.camera_active`/`status.microphone_active`/`status.permissions.*`) and four `StatItem`s reading `status.stats.frames_processed`/`audio_chunks_processed`/`learning_events`/`files_processed`. **Confirmed matching [[dw_controller]]'s `ControllerRuntime.get_stats()` exactly, field for field** — a clean, precise cross-stack confirmation with no discrepancy found.

### `StatItem` / `ActivityIndicator`

Two more small components in this same file, purely presentational — a labelled number and a labelled icon-with-status-dot, respectively.

## Problems (faced by traditional AI systems / LLMs)

Not AI/ML-specific — a plain state-management gap. A value (`agentId`) was correctly resolved once, in a parent component, and a sibling component that needs the same value wasn't given a way to read it — so it re-declares its own, necessarily-static substitute instead.

## Solutions

Not solved in this file — there's no code path here that reaches back for `App.jsx`'s real `agentId` or derived URLs. The fix, if made, is mechanical: thread `agentId`/`BACKEND_URL` through as props from `App.jsx`'s render call, the same way `messages`/`sendMessage` already are, rather than adding a fifth independent hardcoded copy.

## Files Required

- [[wiki/codebase/files/MessageBubble|MessageBubble.jsx]].

## Files Used In

- [[wiki/codebase/files/App|App.jsx]] — rendered in place of the entire chat view when `mode === "controller"`.