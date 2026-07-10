---
type: file
status: ingested
---
# config_loader.py

💡 **Role**: exists purely to satisfy one import. Its own module docstring says so directly: `world_model.py` needs `from ai_core.config_loader import get_section, get_device`, and this file provides exactly those two functions rather than requiring `world_model.py` itself to change. All the codebase's _real_ configuration lives in `config.py`'s `Config` class (not yet given its own file-level page); this is a small, optional YAML/env-var layer sitting alongside it.

## Imports

`os`, `logging`, `torch`, `typing` — all external/stdlib. `yaml` (PyYAML) is soft-imported inside `_load_yaml()` only, wrapped in its own `try`/`except ImportError` — its complete absence just means every `get_section()` call falls through to whatever default the caller supplied.

## Module-level state

- `_yaml_config` — a plain module-level dict, populated once at import time by `_load_yaml()`.

## Functions

- `_load_yaml()` — checks two candidate paths (`py_backend/config.yaml`, `ai_core/config.yaml`) and loads the first one found; silently does nothing if neither exists or PyYAML isn't installed. Called once, unconditionally, at module import time — not lazily on first use.
- `get_section(section, default=None)` — returns `_yaml_config.get(section, default or {})`. The whole "config system" here is really just this one dict lookup with a fallback.
- `get_device()` — three-tier priority: `DW_DEVICE` env var, then a YAML `device` key, then auto-detection (CUDA → MPS → CPU, in that order — MPS checked via `hasattr` first since it's only present in recent torch builds).

No classes, no async functions anywhere in this file.

## Problems (faced by traditional AI systems / LLMs)

A narrow, practical problem rather than a general AI/ML one: a module (`world_model.py`) was written assuming a configuration interface that didn't exist yet on disk, and retrofitting that interface directly into `config.py` would have meant touching a file with much broader blast radius than the two functions actually needed.

## Solutions

A thin adapter module, scoped to exactly the two functions the calling code needs, backed by an entirely optional YAML layer — nothing breaks if `config.yaml` is absent or PyYAML isn't installed, since every path has a hardcoded fallback.

## Files Required

None — this file has no internal project imports.

## Files Used In

- `world_model.py` — the entire reason this file exists; see [[wiki/codebase/files/world_model|world_model.py]] and [[world-model]].