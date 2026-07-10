---
type: file
status: ingested
---
# config.py (two copies: `py_backend/config.py` and `py_backend/ai_core/config.py`)

💡 **Role**: this page exists to document a genuine naming collision rather than force one onto the "exact filename" convention — there are two files named `config.py`, both defining a `Config` class, at `py_backend/config.py` and `py_backend/ai_core/config.py`. **This confirms and resolves `CLAUDE.md`'s own "known open item"**: `AUDIT_REPORT.md`'s duplicate-Config-class finding, flagged there as "not confirmed fixed." It's confirmed now — and it's more interesting than a stale copy-paste: the two files serve two genuinely different runtime contexts, with one real, verified inconsistency between them.

## Why two copies exist at all

`py_backend/main.py`, `agent_spawner.py`, `minecraft_launcher.py`, `auto_connect_system.py`, `chat_launcher.py`, `auto_packager.py`, and `packager.py` all import `from py_backend.config import Config` — every one of these is a server-side or build-time script, consistent with being invoked from the repo root (one level _above_ `py_backend/`), where `py_backend` itself is an importable top-level package.

`py_backend/ai_core/__init__.py` — which runs for every single `ai_core.*` import anywhere in the codebase — imports `from ai_core.config import Config` instead: the bare-path convention, consistent with every other `ai_core/*.py` file's own imports (`from ai_core.brain_core import BrainCore`, and so on, throughout this entire ingest). This is the convention a **packaged agent executable** runs under, where `py_backend/`'s own contents become the executable's root and `ai_core`/`rl` are importable directly, without a `py_backend.` prefix.

So this isn't simply "someone forgot to delete the old one" — `main.py` (the manager, per Devlord's own clarification) and a packaged individual agent genuinely run with different roots, and each needs a `Config` reachable under the import path that context actually uses. Both files are 237 lines, identical except for: the header comment, `PY_BACKEND_DIR`'s calculation (correctly adjusted — `.parent.parent` in the `ai_core/` copy vs. `.parent` in the `py_backend/` copy, since one starts one directory deeper), and every `ai_core.*`/`rl.*` entry in `AGENT_HIDDEN_IMPORTS`/`AGENT_EXCLUDE_MODULES` being bare in one copy and `py_backend.`-prefixed in the other.

## The one real, confirmed inconsistency

Both copies share the identical comment block above `AGENT_EXCLUDE_MODULES`, which explicitly names two things that should be excluded for the same reason: `py_backend.agent_spawner` and `ai_core.agent_spawner` ("spawning belongs to the server... same reason"). **Neither file's actual list matches that comment**:

- `py_backend/ai_core/config.py`'s list has only `'ai_core.agent_spawner'` — `'py_backend.agent_spawner'` is missing entirely, despite being named in the shared comment.
- `py_backend/config.py`'s list has `'py_backend.agent_spawner'` _and_ `'py_backend.ai_core.agent_spawner'` — neither of which is the bare `'ai_core.agent_spawner'` the comment actually names.

Since `packager.py` — the thing that actually invokes PyInstaller with these lists — imports the `py_backend/config.py` copy, that file's version is the operative one for real builds. Given every `ai_core/*.py` module resolves its own sibling imports as bare `ai_core.X` (confirmed throughout this entire ingest pass), a packaged agent executable would resolve a transitive `agent_spawner` import the same way — and `py_backend/config.py`'s exclude list doesn't have an entry that would match that bare form. **Not confirmed as an active bug** — this depends on whether anything bundled into an agent executable actually imports `agent_spawner` transitively, which wasn't traced this pass, and `agent_spawner.py` itself imports `from py_backend.config import Config` (the _other_ convention), suggesting it may not be reachable from inside a packaged agent context at all. Recorded as a real, verifiable inconsistency regardless of whether it's currently load-bearing.

## Everything else is identical

Paths (`BRAINS_DIR`, `UPLOADS_DIR`, `FRONTEND_DIR`, etc.), network/port defaults (`BASE_BACKEND_PORT=11400`), Minecraft/UltimMC settings, safety limits, performance caps, logging format, MinIO/S3 settings, and all four classmethods (`ensure_dirs()`, `validate()`, `get_agent_brain_path()`, `get_agent_upload_dir()`, `get_package_dir()`) are byte-for-byte the same in both copies.

## Files Required

None — neither copy imports anything from elsewhere in this codebase.

## Files Used In

- `main.py`, `agent_spawner.py`, `minecraft_launcher.py`, `auto_connect_system.py`, `chat_launcher.py`, `auto_packager.py`, `packager.py` — all import the `py_backend/config.py` copy.
- `ai_core/__init__.py` — imports the `py_backend/ai_core/config.py` copy, making it live for every `ai_core.*` import anywhere in the codebase.
- Both copies' `AGENT_HIDDEN_IMPORTS`/`AGENT_EXCLUDE_MODULES` lists are what `packager.py` reads (via `PYINSTALLER_HIDDEN_IMPORTS`, a back-compat alias for the same list) to decide what goes into a packaged agent executable.