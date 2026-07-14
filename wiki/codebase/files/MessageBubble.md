---
type: file
status: ingested
---
# MessageBubble.jsx

💡 **Role**: a pure presentational component — one chat bubble, styled by who sent it. No state, no side effects, no API calls. See [[electron-frontend]] for how it fits into the chat view.

## Imports

`framer-motion` (`motion`) — external. No internal imports.

## Props

- `sender` — `"user"` | `"agent"` | `"system"` (or anything else, which falls through to the agent styling as a default).
- `text` — the message content, rendered as-is inside the bubble.

## Component

Picks a Tailwind/DaisyUI chat-bubble class set and alignment based on `sender` (`chat-end` + indigo for `"user"`, `chat-start` + slate for `"agent"`/default, a centered system-style pill for `"system"`), wraps the whole thing in a `framer-motion` entrance animation (fade + slight vertical slide). That's the entire component.

## Problems / Solutions

Not applicable — no logic beyond conditional styling.

## Files Required

None beyond `framer-motion`.

## Files Used In

- [[wiki/codebase/files/App|App.jsx]] — renders one per entry in `messages`.
- [[wiki/codebase/files/ControllerSafety|ControllerSafety.jsx]] — reuses this same component for its own chat sidebar, sharing `App.jsx`'s `messages` state via props.