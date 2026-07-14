---
type: file
status: ingested
---
# App.jsx

💡 **Role**: the whole application's state and orchestration — WebSocket connection and reconnect, chat send/receive, file upload, mode switching (chat/controller), and the two visualizer panels. [[electron-frontend]] already covers this file's role at the narrative level in real depth; this page adds the full mechanical catalog, including the two inline sub-components it defines (`WebAccessManager`, `ActivityBar`) that aren't separate files.

## Imports

`react` (`useState`, `useEffect`, `useRef`, `useLayoutEffect`), `lucide-react` icons, `framer-motion` (`motion`, `AnimatePresence`) — external. [[wiki/codebase/files/MessageBubble|MessageBubble.jsx]], [[wiki/codebase/files/AgentStatus|AgentStatus.jsx]], [[wiki/codebase/files/FileDropZone|FileDropZone.jsx]], [[wiki/codebase/files/ControllerSafety|ControllerSafety.jsx]], [[wiki/codebase/files/WorldModelVisualizer|WorldModelVisualizer.jsx]], [[wiki/codebase/files/MentalMatrixModal|MentalMatrixModal.jsx]] — internal.

## Module-level constants

- `BACKEND_PORT = window.location.port ? Number(window.location.port) - 1 : 11400` — [[wiki/codebase/files/packager|packager.py]] always serves this bundle from `backend_port + 1`, so the page's own URL is enough to derive the backend's port without any config file or env var.
- `BACKEND_URL` / `WS_URL` — both branch on `import.meta.env.DEV` (Vite's own "am I running under `vite`/`npm run dev`" flag). **In dev**: relative paths (`''` for HTTP, `ws://{host}/agent-ws` for the socket), routed through `vite.config.js`'s own dev-server proxy to `127.0.0.1:11400` — confirmed matching that file's proxy config exactly, including the `/agent-ws` → `/ws` path rewrite. **In production** (the packaged bundle): absolute URLs built from `BACKEND_PORT` directly. The in-line FIX comment explains why the branch exists at all: before it, the packaged-launcher scheme was applied unconditionally, so `npm run dev` (Vite itself listening on 8765) computed `BACKEND_PORT = 8764` — nothing listens there, and an absolute URL to the wrong port can't be caught by the dev proxy either, since that only intercepts same-origin relative requests.
- `getOrCreateVisitorId()` / `VISITOR_ID` — a UUID persisted in `localStorage` (falling back to a per-session id if unavailable), sent as `speaker_id` with every chat message. The in-line comment ties this directly to a specific backend feature: `RewardSystem`'s `familiarity_r` term needs a stable per-visitor identity to recognize a repeat visitor across sessions; before this existed, every chat looked identical to the backend and an agent's extraversion trait had no repeat-visitor signal to read as a reward at all. Attributed to the same "Chat & Web GRPO plan" named elsewhere this ingest, on the Python side, in [[web_browser]] and [[cognitive_loop]] — confirms that plan spans both the backend and this frontend, not just one layer.

## Inline sub-components (defined in this file, not separate files)

### `WebAccessManager({ onWebsitesChange, agentId })`

The per-chat browsing allow-list editor. Local state: `websites` (array of `{id, url, display, type, enabled}`), `newUrl`, `isAdding`. A `useEffect` POSTs the enabled subset to `/api/agents/{agentId}/web/allow` whenever `websites` changes — **with an early-return guard, `if (websites.length === 0) return`, that means removing the last website never syncs the empty state to the backend**; the backend keeps whatever non-empty list it last received. `toggleWebsite`/`removeWebsite`/`addWebsite` are plain local-array mutations; the sync effect is what actually reaches the backend.

### `ActivityBar({ activities, theme })`

A collapsible log of the last 20 `activities` entries, icon-coded by `type`. Purely presentational over whatever `App`'s own `activities` state contains — see the note on `App` itself below about how sparse that state actually is in practice.

## `App()` — state

`connected`, `error`, `messages`, `thoughts`, `activities`, `text`, `mode` (`"chat"`/`"controller"`), `theme`, `visualizationData` (defaults `{type: "matrix", label: "Mental Matrix"}`), `allowedWebsites`, `showWebManager`, `brainActive`, `worldModelData`, `showMentalMatrix`, `agentId` (starts `"demo"`, resolved from `GET /status` on mount — the one piece of backend identity that can't be derived from the URL the way the port can). Refs: `wsRef`, `reconnectTimeoutRef`, `shouldReconnectRef`, `messagesEndRef`, `thoughtsEndRef` (the latter two auto-scrolled via `useLayoutEffect`, not `useEffect` — layout-synchronous, so the scroll happens before the browser paints the new message, avoiding a visible jump).

## `App()` — WebSocket

Connects once on mount (`wsRef.current` guard prevents a second connection from a Strict-Mode double-invoke), 5-second auto-reconnect on close, tracked via `shouldReconnectRef` so the cleanup function on unmount can suppress a reconnect that would otherwise race the component's own teardown. Message-type switch, confirmed against the actual backend senders documented elsewhere this ingest:

- `"connected"` — sets connected/clears error.
- `"chat"` — appends to `messages`. This, not `"agent_speech"` below, is what [[cognitive_loop]]'s `_broadcast_speech` actually sends (`type: "chat"`, `from: "agent"`) — confirmed by this file's own in-line comment.
- `"agent_thought"` — appends to `thoughts`, flips `brainActive` true for 2 seconds.
- `"agent_speech"` — handled, but per the comment above, dead code in practice; nothing sends this exact type.
- `"visualization_update"` — sets `visualizationData`, from [[cognitive_loop]]'s `_broadcast_mental_workspace`.
- `"world_model_update"` — sets `worldModelData`, from `agent.py`'s `broadcast_world_model()`. **Confirmed this pass: that function has zero callers anywhere in the repository** — this case is reachable code with no live sender. Since [[wiki/codebase/files/WorldModelVisualizer|WorldModelVisualizer.jsx]] only mounts when `worldModelData` is truthy, the entire panel never appears in the running UI.
- `"activity_update"` — prepends to `activities` (capped at 20), from `agent.py`'s `broadcast_activity()`. **Also confirmed zero callers.** `activities` isn't entirely dead, though — `onFileSend` below pushes its own `file`-type entry directly, client-side, with no backend involvement at all.

## `App()` — functions

- `sendMessage(customText)` — optimistically appends the user's own message locally, `POST`s to `/chat` as `FormData` (`message`, `agent_id`, `speaker_id: VISITOR_ID`, and `allowed_websites` as a JSON-stringified array if any site is enabled). **The WS/HTTP dedup logic, confirmed precisely**: the HTTP response's `d.response` is only ever added to `messages` if `!connected` — the assumption, stated in-line, is that the WebSocket delivers the same reply first (and faster) whenever it's actually up, so adding both would double-post; the HTTP path exists purely as a fallback for a disconnected socket.
- `onFileSend(file, type, sync)` — `POST`s to `/api/upload` as `FormData`. The in-line comment on `sync` is a real, specific API-contract note worth preserving: it has to be a form field, not a query parameter, because `agent.py` reads it via `Form(False)`.
- `toggleTheme`, `dismissError` — trivial state setters.

## `App()` — render

Branches entirely on `mode`: `"controller"` renders [[wiki/codebase/files/ControllerSafety|ControllerSafety.jsx]] in place of everything else, passed `onModeChange`, `messages`, `sendMessage`, `inputText`/`setInputText` (`text`/`setText` under different prop names) — genuinely sharing chat state with the main view, not a fully isolated experience. `"chat"` (default) renders the two-panel layout [[electron-frontend]] already describes: chat + file drop + web access manager on the left, Mental Workspace (a summary view, expandable via "Simulate" into [[wiki/codebase/files/MentalMatrixModal|MentalMatrixModal.jsx]]) + [[wiki/codebase/files/WorldModelVisualizer|WorldModelVisualizer.jsx]] (conditionally) + Cognitive Stream + `ActivityBar` on the right. `MentalMatrixModal` itself is always mounted, visibility controlled by `showMentalMatrix`.

## Problems (faced by traditional AI systems / LLMs)

Two distinct problems, both concretely visible in this one file. First, a deployment-environment problem general to any app that needs to find its own backend without hardcoding an address: the same built bundle has to work whether it's served from a packaged agent's own port, or from Vite's dev server on a completely different port proxying elsewhere — and get this wrong, and _nothing_ works locally even though production is fine (or vice versa), which is exactly what the documented `BACKEND_PORT` fix was responding to. Second: reconciling two delivery channels (WebSocket push, HTTP request/response) for the same logical event (an agent's reply) without either losing messages or duplicating them.

## Solutions

An environment flag (`import.meta.env.DEV`) branching the URL-derivation logic, rather than trying to make one formula correct for both environments, is the real fix for the first problem — confirmed to line up exactly with `vite.config.js`'s own dev-proxy configuration on the other side. The WS/HTTP dedup logic (add the HTTP reply only when the socket is down) is a plain, working answer to the second — not sophisticated, but correct for the actual failure mode it's guarding against (a disconnected socket, not a slow one).

## Files Required

- [[wiki/codebase/files/MessageBubble|MessageBubble.jsx]], [[wiki/codebase/files/AgentStatus|AgentStatus.jsx]], [[wiki/codebase/files/FileDropZone|FileDropZone.jsx]], [[wiki/codebase/files/ControllerSafety|ControllerSafety.jsx]], [[wiki/codebase/files/WorldModelVisualizer|WorldModelVisualizer.jsx]], [[wiki/codebase/files/MentalMatrixModal|MentalMatrixModal.jsx]].

## Files Used In

- [[wiki/codebase/files/main_jsx|main.jsx]] — the sole render target.
- Talks to `agent.py`'s FastAPI app (`/status`, `/chat`, `/api/upload`, `/api/agents/{id}/web/allow`, `WS /ws`) — not [[main]]'s app; per [[agent]]'s own already-documented routing, this frontend is served by and talks to the per-agent backend, not the central manager.
