---
type: file
status: ingested
---
# main.jsx

💡 **Role**: the React entry point — nine lines, nothing more. Mounts `<App />` into `#root` via React 19's `createRoot`, wrapped in `<React.StrictMode>`. See [[electron-frontend]] for the surrounding system this bootstraps into.

## Imports

`react`, `react-dom/client` (`ReactDOM`), `./App.jsx`, `./index.css`.

## Problems / Solutions

Not applicable — this file has no logic to speak of.

## Files Required

- [[wiki/codebase/files/App|App.jsx]] — the entire application.

## Files Used In

- `index.html` (not read this pass) — the Vite entry HTML presumably loads this as a module script, per standard Vite scaffolding; not independently confirmed.