---
type: file
status: ingested
---
# MentalMatrixModal.jsx

💡 **Role**: the full-screen modal wrapper around [[wiki/codebase/files/MentalMatrixSimulator|MentalMatrixSimulator.jsx]] — the "Simulate" button's target. Owns the export/import UI around the simulator; the simulator itself is a separate file.

## Imports

`react` (`useState`), `framer-motion` (`motion`, `AnimatePresence`), `lucide-react` icons — external. [[wiki/codebase/files/MentalMatrixSimulator|MentalMatrixSimulator.jsx]] — internal.

## Props

- `isOpen` — controls mounting; the component returns `null` outright when false, rather than hiding via CSS.
- `onClose` — close handler.
- `agentId` — passed straight through to `MentalMatrixSimulator` as its own `agentId` prop, which — per that component's own page — never actually uses it.

## State

- `simulationState` — set via the `onSimulationEvent` callback passed to `MentalMatrixSimulator`. **Fixed as of the 2026-07-16 pull**: previously always stayed `null`, since `MentalMatrixSimulator` accepted that callback but never called it — now genuinely populated, since that component calls it from a reporting effect keyed on its own object list. See [[wiki/codebase/files/MentalMatrixSimulator|MentalMatrixSimulator.jsx]] for the fix itself.

## Component

A fade-only entrance animation with a specific, documented reason in its own in-line comment worth preserving: no `scale` transform on entry, because `scale(0.9)` collapses the container to near-0px before Three.js can measure its size on mount, producing a black viewport — fade-only sidesteps the issue entirely rather than working around it. Two footer buttons: "Load State" has no `onClick` handler at all — not a stub, not wired to anything, just decorative, unaffected by the 2026-07-16 fix — and "Dump Buffer" (`handleExport`) downloads `simulationState` as a JSON file. **This button is fixed as of that same pull**: previously permanently a no-op (its own early-return guard on a `simulationState` that never left `null`); now genuinely exports real object data, since `simulationState` gets populated for the first time.

## Problems (faced by traditional AI systems / LLMs)

Not AI/ML-specific — the interesting problem here is a real, narrow browser-rendering gotcha: combining a CSS scale-transform entrance animation with a Three.js canvas that measures its own container on mount. The two don't compose safely without extra work neither this file nor the simulator does.

## Solutions

Dropping the scale transform (fade-only entrance) sidesteps the measurement-race problem entirely rather than solving it — a pragmatic, working choice given the alternative (delaying Three.js init until after the animation settles) would add real complexity for one modal.

## Files Required

- [[wiki/codebase/files/MentalMatrixSimulator|MentalMatrixSimulator.jsx]].

## Files Used In

- [[wiki/codebase/files/App|App.jsx]] — rendered unconditionally (always in the DOM), visibility controlled by `isOpen`/`showMentalMatrix` state, opened via the "Simulate" button in the Mental Workspace panel or the button inside it.