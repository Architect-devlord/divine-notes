---
type: file
status: ingested
---
# FileDropZone.jsx

💡 **Role**: a drag-and-drop file upload zone with a "Synchronous Processing" toggle. Owns its own small piece of state; the actual upload request is delegated entirely to a callback prop, not made here.

## Imports

`react` (`useState`) — external. No internal imports.

## Props

- `onFileSend(file, type, sync)` — called on drop or manual file pick; this component performs no `fetch` itself. See [[wiki/codebase/files/App|App.jsx]] for the actual implementation.

## State

- `isDragging` — toggles the drop-zone's highlighted styling during a drag-over.
- `sync` — the "Synchronous Processing" checkbox value, passed straight through to `onFileSend`.
- `lastFile` — the most recently sent filename, shown as a 3-second success confirmation before clearing itself via `setTimeout`.

## Component

Standard HTML5 drag-and-drop handlers (`onDragOver`/`onDragLeave`/`onDrop`), each preventing default; a hidden `<input type="file">` triggered by clicking the zone, for non-drag selection. `type` is inferred from the file's extension/MIME type into a coarse category (`image`/`document`/`code`/`other`, exact mapping not central to this component's own logic since it's just forwarded to the callback).

## Problems / Solutions

Not applicable — the interesting logic (what actually happens with the file) lives in the parent's `onFileSend` implementation, not here.

## Files Required

None beyond React itself.

## Files Used In

- [[wiki/codebase/files/App|App.jsx]] — rendered once, above the message input, with `onFileSend` wired to a `POST /api/upload` call. That page's own comment is worth repeating here since it explains a real, non-obvious API contract: `sync` has to arrive as a `FormData` field, not a query parameter, because `agent.py` reads it via `Form(False)`.