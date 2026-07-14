---
type: file
status: ingested
---

# WorldModelVisualizer.jsx

💡 **Role**: a 2-D canvas node-link graph for `world_model_update` data. [[electron-frontend]] already covers this component's intended purpose and its two already-fixed rendering bugs in detail — this page adds the mechanical catalog and **a confirmation that upgrades that page's own "wasn't traced this pass" flag to a settled fact**: `agent.py`'s `broadcast_world_model()`, the only thing that would ever send this component real data, has zero callers anywhere in the repository. Combined with how `App.jsx` mounts this component (only when `worldModelData` is truthy, which in turn is only ever set inside that same WebSocket handler), the practical result is stronger than "shows stale data" — this panel never appears in the running UI at all under the current backend.

## Imports

`react` (`useState`, `useRef`, `useEffect`, `useCallback`) — external only. No internal imports; this component computes its own layout rather than importing a graph library.

## Props

- `data` — the `world_model_update` payload: `{nodes: [...], stats: {...}}`. Never populated in practice (see above).
- `onPredictionRequest` — accepted, never called anywhere in this file, confirmed by direct read. Same pattern as [[wiki/codebase/files/MentalMatrixSimulator|MentalMatrixSimulator.jsx]]'s `onSimulationEvent`.

## State

- `canvasSize` — kept in sync with the container via `ResizeObserver`; this is the fix for the already-documented blurry-canvas bug (a hardcoded 800×600 canvas that CSS was stretching).
- `selectedNode` — set on click via a hit-test against node positions.
- `hoveredNode` — declared, and read inside the draw logic (drawing a highlight outline when `selectedNode === i || hoveredNode === i`), but **nothing anywhere in this file ever calls its setter**. There's no `onMouseMove` handler at all — only clicks update selection state, so the hover half of that condition can never be true. Half-wired: the state and the render logic both exist, the event handler that would drive it doesn't.
- `showPredictions` — toggled by a button (label/color flips between "On"/"Off"), included in the redraw effect's dependency array, but never actually referenced inside the draw function itself. Toggling it triggers a redraw that produces identical output either way.
- `animFrame` — drives a continuous pulse animation on node size via `requestAnimationFrame`, independent of any real data change.

## Functions

- `drawNetwork(ctx, ...)` — the actual rendering: computes a simple force-free circular/radial layout from `data.nodes`, draws edges then nodes, color-codes by `type` (`vision`/`action`/`reward`/`state`), sizes and pulses by `activation`.
- `handleCanvasClick(e)` — hit-tests the click point against drawn node positions (using live `canvasSize`, not the old hardcoded dimensions — the second already-documented fix), sets `selectedNode`.
- Stats readout — reads `data.stats` directly (buffer size, health/status fields) alongside the canvas.

## Problems (faced by traditional AI systems / LLMs)

Not AI/ML-specific — this file's own history is a plain rendering-correctness story: a canvas element's intrinsic pixel dimensions and its CSS-rendered size are two different things, and code that assumes they match (hardcoded dimensions for both drawing and hit-testing) breaks the moment CSS stretches the element to fit its container.

## Solutions

`ResizeObserver` keeping `canvasSize` state in sync with the container's actual layout size, used consistently for both the draw calls and the click hit-test, is the correct fix for exactly that class of bug — and it's already applied here, per the component's own header comment documenting the fix. The `hoveredNode`/`showPredictions` gaps found this pass aren't solved in this file; they're just unfinished — render logic and toggle UI built ahead of the event wiring or the backend feature that would make them matter.

## Files Required

None beyond React itself.

## Files Used In

- [[wiki/codebase/files/App|App.jsx]] — rendered only when `worldModelData` is truthy, inside the Mental Workspace panel.