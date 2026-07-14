---
type: file
status: ingested
---
# MentalMatrixSimulator.jsx

💡 **Role**: a genuinely well-built, entirely local Three.js physics sandbox — orbit camera, drag-to-move objects in full 3D, add primitives, import `.glb`/`.gltf`/`.obj` models, basic gravity-and-bounce physics. [[electron-frontend]] already covers the headline finding — confirmed by Devlord and independently on this pass — that this component has no connection to the backend's actual Mental Matrix system ([[world-model]]'s `MentalMatrixSimulation`/`MentalMatrixService`/`MentalMatrixWebSocketManager`) despite the shared name and near-identical premise. This page adds the mechanical catalog and one small addition to that finding: it isn't only `onSimulationEvent` that goes unused — the `agentId` prop this component accepts is never referenced anywhere in its body either, confirmed by a full read. Every object here exists only in that browser tab and is gone on close or reset.

## Imports

`react` (`useState`, `useEffect`, `useRef`, `useCallback`), `three` (`THREE`), `three/examples/jsm/loaders/GLTFLoader`, `three/examples/jsm/loaders/OBJLoader`, `lucide-react` icons — all external. No internal imports.

## Props

- `agentId` — accepted, never referenced anywhere in the component body.
- `onSimulationEvent` — accepted, never called anywhere in the component body. See [[wiki/codebase/files/MentalMatrixModal|MentalMatrixModal.jsx]] for the downstream consequence (its "Dump Buffer" export permanently no-ops as a result).

## Refs

`containerRef`, `sceneRef`, `cameraRef`, `rendererRef`, `animFrameRef`, `initDoneRef` (guards Three.js init to run exactly once), `raycasterRef`, `meshMapRef` (id → `THREE.Object3D`), `physicsRef` (id → `{velocity, useGravity, mass}`), `orbitRef` (`{theta, phi, radius}`, the camera's spherical-orbit state), `mouseRef` (drag/orbit interaction state, including a per-grab `dragPlane`/`dragOffset`), `isRunningRef`/`timeScaleRef` (mirrors of the matching state, kept in refs so the animation loop's closure reads live values without needing to be recreated every render).

## State

`isRunning`, `simulatedObjects` (the list backing the UI's object panel — `meshMapRef`/`physicsRef` hold the real Three.js/physics data; this is just the React-visible mirror), `selectedId`, `showGrid`, `timeScale`, `showDropdown`, `interactMode` (`"orbit"`/`"drag"`, drives cursor style), `importStatus`/`importMsg` (transient, self-clearing via `setTimeout`), `fileInputRef`.

## Key functions

- **Three.js init** (`useEffect`, deferred) — doesn't initialize immediately on mount; tries once with the container's current size, then watches via `ResizeObserver` and initializes on the first nonzero size it sees. This is the correct answer to exactly the same category of problem [[wiki/codebase/files/MentalMatrixModal|MentalMatrixModal.jsx]]'s own fade-only-entrance fix addresses (a container that's 0×0 until a parent animation finishes) — here solved properly via `ResizeObserver` rather than avoiding the animation that causes it. Sets up scene/camera/renderer (shadows enabled, ACES tone mapping), lighting (ambient + a shadow-casting directional sun + a dim fill light), a grid + ground plane, mouse handlers, and the animation loop, then returns a cleanup that removes listeners, cancels the frame, and disposes the renderer.
- `applyOrbit()` — converts the spherical `orbitRef` state into a camera position via `sin`/`cos`, always looking at the origin.
- `raycastObjects(x, y)` — screen coordinates → normalized device coordinates → `THREE.Raycaster` → walks up each hit mesh's ancestor chain to find which tracked top-level object (from `meshMapRef`) it belongs to, since a hit is usually on a child mesh, not the registered root.
- **Mouse handlers** — `onMouseDown` raycasts; a hit switches to drag mode (suspends that object's gravity for the duration, builds a camera-facing plane through the exact hit point via `makeCameraPlane()` so dragging moves the object in all three axes rather than being constrained to one), a miss switches to orbit mode. `onMouseMove` either updates the dragged object's position (by re-intersecting the mouse ray with the stored drag plane) or adjusts orbit angles. `onWheel` adjusts orbit radius (zoom), clamped `[3, 120]`.
- **Animation loop** (`animate`, `requestAnimationFrame`) — when `isRunningRef.current`, steps simple gravity + bounce physics (velocity integration, floor collision at `y < 1` with a 0.55 restitution coefficient and lateral damping) for every non-dragged object, then renders every frame regardless of `isRunning` (so orbit/drag interaction stays live even while physics is paused).
- `registerObject(root, type, colorHex)` — the common tail for both `addObject()` and `handleFileImport()`: enables shadows on every mesh in the subtree, adds to the scene, generates a random id, registers into `meshMapRef`/`physicsRef` (gravity on by default), and mirrors into `simulatedObjects` state.
- `addObject(type)` — cube/sphere/cylinder primitives, random color and spawn position (elevated, so it visibly falls into place under gravity).
- `handleFileImport(e)` — `.glb`/`.gltf` via `GLTFLoader`, `.obj` via `OBJLoader`; on success, auto-scales the imported model to fit a ~10-unit box and re-centers it to sit on the ground, with a random horizontal offset so multiple imports don't stack exactly on top of each other. Shows a self-clearing loading/success/error indicator throughout.
- `removeObject(id)` / `resetSimulation()` — proper Three.js cleanup on removal (disposes geometry and every material on every mesh in the subtree, not just removing from the scene graph) rather than just detaching and leaking GPU resources; reset clears every object and returns the camera to its default orbit.
- `updateVelocity(id, axis, val)` / `applyImpulse(id, vector)` — the properties panel's direct velocity edit and the two impulse buttons ("Launch" straight up, "Scatter" randomized horizontal + up).

## Problems (faced by traditional AI systems / LLMs)

Not AI/ML-specific — the interesting problems here are general 3D-web-app ones, solved competently: measuring a container that isn't sized yet on mount, dragging an object in full 3D from a 2D mouse position (constraining to a screen-facing plane rather than a fixed world plane, so the drag direction always matches what the camera is actually looking at), and not leaking GPU memory when objects are removed.

## Solutions

`ResizeObserver`-deferred initialization, a camera-facing drag plane rebuilt per-grab, and explicit geometry/material disposal on removal are all correct, specific answers to those three problems respectively — this is solid engineering in isolation. None of it touches the actual gap this component has, which isn't a bug in what it does, but a disconnection from what it was apparently meant to connect to: real Mental Matrix data from the backend.

## Files Required

None beyond `three` and its loaders.

## Files Used In

- [[wiki/codebase/files/MentalMatrixModal|MentalMatrixModal.jsx]] — the sole renderer, passing `agentId` and `onSimulationEvent` (neither used, per above).