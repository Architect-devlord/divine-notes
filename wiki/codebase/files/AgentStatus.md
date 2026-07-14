---
type: file
status: ingested
---

# AgentStatus.jsx

💡 **Role**: another pure presentational component — the connection/activity status pill in the chat header. No state, no side effects, no API calls.

## Imports

`framer-motion` (`motion`) — external. No internal imports.

## Props

- `connected` — boolean, drives the dot color (emerald/rose) and label ("Online"/"Offline").
- `active` — boolean, drives a second "Thinking"/"Idle" indicator, driven by `App.jsx`'s `brainActive` state (set true for 2 seconds on every `agent_thought` WebSocket message).

## Component

Two small indicator groups side by side — a pulsing dot + label for connection state, a `framer-motion`-animated pulse icon + label for activity state. Purely reactive to its two boolean props; no logic of its own beyond conditional styling.

## Problems / Solutions

Not applicable.

## Files Required

None beyond `framer-motion`.

## Files Used In

- [[wiki/codebase/files/App|App.jsx]] — rendered once in the chat header, given `connected` and `brainActive`.