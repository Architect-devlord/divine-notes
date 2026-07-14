---
type: file
status: ingested
---
# agents_json_manager.py

💡 **Role**: a pure compatibility shim, 33 lines, nothing more. Its own docstring states plainly that `mc_uuid.AgentNameManager` is the single source of truth for `agents.json`, and this module exists only so old or Java-facing import paths (`from utils.agents_json_manager import AgentsJsonManager`) keep resolving to that same implementation rather than diverging into a second one. Explicitly warns against adding logic here: "add it to `AgentNameManager` in `mc_uuid.py`."

## Imports

`py_backend.utils.mc_uuid` — `AgentNameManager` (aliased `AgentsJsonManager` at import time), `get_minecraft_uuid`.

## Module-level

- `AgentsJsonManager = AgentNameManager` — not a subclass, not a wrapper class, literally the same class object under a second name.
- `_manager: AgentsJsonManager | None = None` — a private module-level singleton slot.
- `get_manager() -> AgentsJsonManager` — lazily constructs and returns the one shared instance. **Confirmed as the exact function [[main]]'s own `get_agents_manager` import resolves to** — that page already documented `main.py` importing this under an alias; this file is the other end of that import.

## Problems / Solutions

Not applicable in the usual sense this wiki uses these sections for — there's no AI/ML problem here, just an import-compatibility one, and the file's own docstring already states the solution as directly as it could be stated: keep one real implementation, re-export it under whatever name old call sites expect, and refuse to let a second implementation grow here by accident.

## Files Required

- `py_backend/utils/mc_uuid.py` — `AgentNameManager`, `get_minecraft_uuid`. The next file in this batch.

## Files Used In

- [[main]] — `get_agents_manager` (aliased on import) is this file's `get_manager()`, confirmed.