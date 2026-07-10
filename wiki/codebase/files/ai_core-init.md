---
type: file
status: ingested
---
# **init**.py (`ai_core/__init__.py`)

💡 **Role**: the package entry point — runs before any other `ai_core.*` import anywhere in the codebase, and is what makes `from ai_core.config import Config` (the bare-path convention, see [[wiki/codebase/files/config-two-copies|config-two-copies]]) the live copy in a packaged-agent context. Re-exports the whole public surface so callers can `from ai_core import NPCAgent` instead of reaching into individual submodules.

## Imports

`from ai_core.config import Config` — the bare-path copy, confirming which of the two `config.py` files is authoritative when `ai_core` is imported as a top-level package (see [[wiki/codebase/files/config-two-copies|config-two-copies]]). `from ai_core.communication_protocol import PerceptionFrame, ActionFrame, ChatFrame` (dataclasses only, not the handler functions — see [[communication_protocol]]). `from py_backend.utils.validation import ...` — the one full `py_backend.`-prefixed import in this otherwise bare-path file, since `utils/` sits outside `ai_core/` with no bare-path equivalent reachable from here. Then, in dependency order (each avoiding importing something that would import it back): `Personality`, `EmotionSystem`, `Memory`/`EpisodicMemory`, `BrainCapsule` → `VisionAdapter`, `AudioProcessor`, `ActuatorManager` → `WorldModel`, `RewardSystem`, `BrainCore` → `CognitivePlanner`, `CognitiveLoop` → `NPCAgent` last, since it constructs all of the above.

## Fixed bug, recorded in the file's own comment

An earlier version imported validation helpers via `from utils import validation` — a bare `utils` import that doesn't resolve to anything in this codebase (there's no top-level `utils` package; it's `py_backend/utils/`). Fixed to the fully-qualified `from py_backend.utils import validation`.

## Module-level content

- `__all__` — the complete list of names this package re-exports; matches the import list above, alphabetized by category rather than declared in file order.
- `__version__` — a plain version string.

No classes, no functions, no async code — this file is purely import wiring.

## Problems (faced by traditional AI systems / LLMs)

Any package assembled from many interdependent modules risks circular imports if the entry point imports them in the wrong order — module A needing module B, which needs something A already partially defined, deadlocking the import. A second, narrower problem: a bare-name import (`from utils import validation`) that happens to work in one execution context (say, a REPL run from inside `utils/`'s own parent directory) can silently fail in every other context that doesn't share that exact working directory.

## Solutions

Explicit ordering — storage/state classes first (nothing to import from elsewhere), sensory/actuator classes next (only need the state classes), learning/cognition classes after that (need both), `NPCAgent` last (needs everything) — sidesteps the circular-import risk by construction rather than by detecting and breaking a cycle after the fact. The bare-import bug gets the direct fix: use the fully-qualified path that's correct regardless of working directory.

## Files Required

Every other file in `ai_core/`, `communication_protocol.py`'s dataclasses, and `py_backend/utils/validation.py` — this file's entire job is importing them in the right order.

## Files Used In

Implicitly, everywhere — this file's top-level code runs as a side effect of any `import ai_core` or `from ai_core.X import Y` anywhere in the codebase.