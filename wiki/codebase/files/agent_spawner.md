---
type: file
status: ingested
---
# agent_spawner.py

💡 **Role**: creates `NPCAgent` instances and, optionally, the real Minecraft client processes behind them — a base spawner that works chat-only, and a UltimMC-automated subclass that also handles the account/instance/mod-install/launch chain end to end.

## Imports

`threading`, `time`, `subprocess`, `socket`, `pathlib.Path`, `typing`, `numpy`, `logging` — external/stdlib. `from py_backend.config import Config` (the `py_backend.`-prefixed copy — see [[wiki/codebase/files/config-two-copies|config-two-copies]]), `from ai_core.agent import NPCAgent` (bare path), `from ai_core.personality import GenderType, assign_npc_gender, assign_god_gender` (**corrected, 2026-07-05** — an earlier version of this page wrongly attributed these to a nonexistent `ai_core.mc_gender` module; see [[wiki/codebase/files/personality|personality.py]], which is where they actually live), and `get_minecraft_uuid` — a mix of both import conventions in one file. Two more, lazy, inside `UltimMCAgentSpawner.__init__()` and its methods: `from py_backend.minecraft_launcher import UltimMCLauncher, MultiAgentLauncher` and `from py_backend.utils.mc_uuid import AgentNameManager`.

## Classes

- **`MinecraftClientProcess`** (dataclass-like) — a thin handle: `agent_id`, the `subprocess.Popen` handle, `port`, `server_addr`, plus `is_alive()`/`kill()`.
- **`AgentClientManager`** — manages the actual Minecraft client subprocesses (not the `NPCAgent` Python objects themselves — that's `AgentSpawner`'s job). `allocate_port()` scans upward from `Config.BASE_BACKEND_PORT` for a free port via a real `socket.bind()` probe, not just an incrementing counter — avoids handing out a port something else already grabbed. `spawn_client()` launches the client jar as a subprocess with the standard `-Ddw.agentId`/`-Ddw.backend`/`-Ddw.server` system properties every launcher in this codebase uses; falls back to chat-only (a `None` client) if no jar is configured, rather than failing the whole spawn. `_tail_logs()` runs on its own daemon thread per client, piping subprocess output into this module's logger rather than letting it vanish or block the main process. `kill_client()`/`cleanup_all()` for teardown.
- **`AgentSpawner`** — the base spawner. `GOD_CONFIGS` (class constant) is the actual source of the six gods' default memory allocation and starting personality traits (matching the same six types documented throughout [[god-abilities]] and the design wiki) — worth knowing this is where those specific numbers are defined, not `agent.py` or the design docs. `spawn_npc()`/`spawn_god()` both immediately call `_save_brain()` right after construction — the class's own docstring explains why: so a `brain.pcap` exists on disk without a race, for whatever packaging step runs next to find. `despawn_agent()` saves before removing, not after — a despawned agent's final state is never lost to a crash between removal and save. `cleanup_all()` saves every agent first, _then_ despawns each — deliberately two full passes rather than one combined pass, so one agent's despawn failure can't prevent another agent's brain from being saved first.
- **`UltimMCAgentSpawner`** (extends `AgentSpawner`) — adds full automation: creates an offline Minecraft account with a real UUID, a Forge 1.20.1 instance, installs both mods, and launches via UltimMC with the same system properties, optionally headless via `xvfb-run`. Its own docstring is explicit about a related class it is _not_: [[auto_packager]]'s `EnhancedAgentSpawner` uses this class as its UltimMC delegate and adds packaging on top — the two are siblings in a composition relationship, not a subclass relationship, confirmed directly from both sides now that [[auto_packager]] has been read. `spawn_npc()`/`spawn_god()` both check `ultimmc_available` first and fall through to the base class's plain behavior if UltimMC isn't found, and separately fall back again (with a warning) if the UltimMC-specific setup step fails partway through — two independent fallback points, not one.

No async methods anywhere in this file — process management here is all synchronous `subprocess`/`threading`, not `asyncio`.

## Problems (faced by traditional AI systems / LLMs)

Automating an entire external application's install-and-launch chain (account creation, instance setup, mod installation, process launch) is inherently the kind of multi-step process where any single step can fail for reasons outside the calling code's control (a missing launcher install, a network hiccup during mod download) — a spawner that only has one all-or-nothing path either crashes on every partial failure or silently produces a broken agent.

## Solutions

Layered fallback rather than one path: no UltimMC at all falls back to the base spawner's plain chat-only behavior; UltimMC present but setup failing for this specific agent falls back the same way, just later in the process. Either way, the caller gets a working `NPCAgent` back — degraded (chat-only instead of embodied), never a hard failure, unless something more fundamental (like the base spawn itself) breaks.

## Files Required

- `config.py` (the `py_backend/config.py` copy) — see [[wiki/codebase/files/config-two-copies|config-two-copies]].
- `agent.py` — `NPCAgent` (see [[agent]] and [[agent-runtime]]).
- [[wiki/codebase/files/personality|personality.py]] — `GenderType`, `assign_npc_gender()`, `assign_god_gender()` (see the correction note above — not a separate `mc_gender.py`).
- `minecraft_launcher.py` / `py_backend/utils/mc_uuid.py` — lazy imports inside `UltimMCAgentSpawner`, neither yet given its own page.

## Files Used In

- [[auto_packager]] — `EnhancedAgentSpawner` composes `UltimMCAgentSpawner` as its delegate, per this file's own docstring; confirmed directly, both sides agreeing, now that [[auto_packager]] has been read.
- Presumably `main.py`, as the actual agent-manager process — not directly confirmed this pass.