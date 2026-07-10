---
type: file
status: ingested
---
# logger_setup.py

💡 **Role**: one shared logging configuration function, so every module across the codebase gets consistently-formatted output (and, for the root/agent logger specifically, both console and rotating file output) without each file configuring its own handlers.

## Imports

`logging`, `logging.handlers`, `os`, `pathlib.Path` — all stdlib. No internal project imports.

## Functions

- `setup_logger(name, log_file=None, level=None, console=True)` — returns a configured logger. Reads a level from the `DW_LOG_LEVEL` env var if `level` isn't given explicitly (defaulting to `INFO`). Guards against duplicate handlers on repeated calls with the same logger name (checks `logger.handlers` before adding new ones — a real, common logging pitfall when a module's top-level `setup_logger()` call runs more than once, e.g. via reload). File output, when requested, uses a `RotatingFileHandler` (5MB per file, 3 backups) rather than a plain unbounded file, and creates the parent directory if it doesn't exist.
- `get_agent_logger(agent_id)` — a convenience wrapper: `setup_logger()` with the log file pre-routed to `Config.LOGS_DIR / f"{agent_id}.log"` (see [[wiki/codebase/files/config-two-copies|config-two-copies]]).

No classes, no async functions.

## Problems (faced by traditional AI systems / LLMs)

A multi-module codebase where each file configures its own logging independently tends to produce inconsistent formats, duplicate handlers on re-import, and unbounded log files that eventually fill a disk.

## Solutions

One shared setup function, called from wherever a logger is needed, with duplicate-handler guarding and a rotating file size cap built in once rather than reimplemented per caller.

## Files Required

- `config.py` (either copy — see [[wiki/codebase/files/config-two-copies|config-two-copies]]) — `Config.LOGS_DIR`, used only by `get_agent_logger()`.

## Files Used In

Broadly, across the codebase, wherever a module wants a configured logger rather than the bare `logging` module — not exhaustively traced call-site by call-site this pass.