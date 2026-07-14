---
type: file
status: ingested
---

# auto_connect_system.py

💡 **Role**: `AutoConnectSystem` — on server startup, is supposed to discover every already-packaged agent with `auto_connect: true` in its `config.json`, launch each one, and track their connections back via `/api/player_event`. `integrate_with_backend(app, agent_manager)` is `main.py`'s one wiring point into this file — confirmed as the single call [[main]] makes into it, and this page closes a note that page's own routes section left open ("not yet confirmed what routes it adds beyond what's catalogued here"): four, listed below.

**The headline finding: the startup auto-launch this file exists to do cannot currently fire, for anyone, ever — confirmed by direct cross-file evidence, not inferred.** `scan_agents_folder()` globs `self.agents_folder.glob("DW_*")`. `AutoConnectSystem` is constructed with `agents_folder=str(Config.NPC_APPLICATIONS_DIR)` (`integrate_with_backend()`, confirmed). [[wiki/codebase/files/packager|packager.py]]'s actual, current agent-directory construction — `agent_dir = self.output_dir / agent_id`, `self.output_dir` defaulting to that exact same `Config.NPC_APPLICATIONS_DIR` — produces directories named plainly `{agent_id}`, no prefix, confirmed twice more by explicit in-line comments at the real construction sites ("no `DW_` prefix — matches `main.py` exe_path lookup", "no `DW_` prefix") and independently confirmed a third way by `main.py`'s own matching lookup carrying the identical comment about `packager.py`. A glob for `DW_*` against a folder of plainly-named subdirectories matches nothing. `scan_agents_folder()` runs twice on every server start — once synchronously inside `AutoConnectSystem.__init__()` (a documented `FIX 8`, "scan on construction"), again inside `main.py`'s `lifespan()` startup block — and returns `[]` both times regardless of how many agents actually exist. `launch_all_agents()` is gated behind `if _agents:` in `lifespan()` and never runs.

This isn't a typo in isolation — it's a stale convention that survived in three places while the real one changed in a fourth. [[wiki/codebase/files/packager|packager.py]]'s own module-level docstring (its very first comment block) and a second in-line comment elsewhere in that same file both still describe agents as `DW_{agent_id}` — the old convention, sitting right next to the "no `DW_` prefix" comments explaining that it changed. This file's glob is the one place that old convention never got updated to match.

## Imports

`asyncio`, `json`, `logging`, `threading`, `time`, `pathlib.Path`, `typing` (`Dict`, `List`, `Optional`, `Set`) — external/stdlib. [[config-two-copies]] (`py_backend.config.Config`) at module level. Lazy, inside methods: `py_backend.utils.mc_uuid.AgentNameManager` (twice — `_launch_agent()`'s Minecraft-availability check, and `integrate_with_backend()`'s server-info read), `ai_core.agent.NPCAgent` (twice — `_launch_agent()`'s NPC branch, and `spawn_god_agent()`), `py_backend.chat_launcher.start_chat_interface` (`_launch_frontend_only()` — **partially confirms, doesn't fully resolve, [[main]]'s existing "presumed caller… not yet confirmed" note about `chat_launcher.py`**; this file does import a `start_chat_interface` from that module, but `chat_launcher.py` itself is still unread, so whether that's a re-export of `main.py`'s own function of the same name or a distinct one isn't yet confirmed either way), `fastapi.HTTPException` (imported inside `integrate_with_backend()`, never actually raised anywhere in that function — dead import).

## Classes

### `AutoConnectSystem`

- `__init__(agents_folder, server_addr="127.0.0.1:25565")` — stores the folder/server, three pieces of state (`required_agents`, `connected_agents`, `connection_callbacks`), and — per its own `FIX 8` comment — calls `self.scan_agents_folder()` immediately, so `required_agents` is populated before any caller has to remember to scan first.
- `scan_agents_folder()` — the method carrying the bug above. Globs `DW_*`, reads each `config.json`, keeps only entries with `auto_connect: true`, builds a `required_agents` list and a fresh `asyncio.Event` per agent in `connection_callbacks`.
- `launch_all_agents(spawner)` _(async)_ — `asyncio.gather()`s `_launch_agent()` for every required agent, `return_exceptions=True` so one failure doesn't cancel the rest, logs an ok/fail count, returns whether all succeeded.
- `_launch_agent(spawner, agent_info)` _(async)_ — checks Minecraft availability via `AgentNameManager.get_minecraft_path()` first; if absent, falls straight to `_launch_frontend_only()`. If present: for `agent_type == "npc"`, constructs `NPCAgent(agent_id=..., autonomous=True, mode="minecraft")` **directly** — not through `spawner.spawn_npc()`. For a god type, calls the module-level `spawn_god_agent(spawner, god_type=..., agent_id=...)` below — which _also_ never touches `spawner`. **A second, independent issue, currently moot only because the bug above means this code path never runs**: `spawner` is threaded all the way from `launch_all_agents(spawner)` down through both branches of this method without ever being called. [[auto_packager]]'s `EnhancedAgentSpawner.spawn_npc()`/`spawn_god()` are what actually run `_post_spawn()` (breeding attachment, auto-packaging queue, documented on that page) — neither runs for an agent launched down this path, so even if the `DW_*` bug were fixed, agents auto-launched on startup would come up without breeding attachment or a fresh packaging queue entry, unlike agents spawned through the normal `/api/agents/spawn_single` or `/api/gods/spawn` routes. Any exception during Minecraft launch, or `agent` coming back falsy, also falls to `_launch_frontend_only()`.
- `_launch_frontend_only(agent_id, agent_info)` _(async)_ — resolves a brain path (from the packaged config, falling back to `Config.get_agent_brain_path()`), then runs `chat_launcher.start_chat_interface(agent_id, brain_str, config)` on a background daemon thread (not awaited — fire-and-forget from this method's perspective, though the thread itself blocks on serving).
- `wait_for_all_connections(timeout=60)` _(async)_ — races each agent's own `asyncio.Event.wait()` against a single shared timeout task via `asyncio.wait(FIRST_COMPLETED)`, cancels whichever didn't win, reports which agent IDs are still missing on timeout.
- `mark_connected(agent_id)` — sets the matching event and adds to `connected_agents`; a no-op if the ID isn't one this instance is tracking. **Called from `main.py`'s `/api/player_event` route directly** (`event == "connected"` branch) — confirmed by direct read; not previously mentioned on [[main]], added there now.
- `is_all_connected()` / `get_status()` — plain reads over the tracking state.

## Module-level functions

- `integrate_with_backend(app, agent_manager)` — reads server address from `agents.json` via `AgentNameManager.get_server_info()`, falling back to `Config.DEFAULT_SERVER` on any failure; constructs the **one** `AutoConnectSystem` instance for the process and stores it at `app.state.auto_connect`; registers four routes:
    - `POST /api/autoconnect/scan` — re-runs `scan_agents_folder()`, returns the count and list found (currently always `0`/`[]`, per the bug above).
    - `POST /api/autoconnect/launch` — runs `launch_all_agents(agent_manager.spawner)` on demand.
    - `POST /api/autoconnect/wait?timeout=60` — runs `wait_for_all_connections()`, returns full status.
    - `GET /api/autoconnect/status` — plain status read.
- `spawn_god_agent(spawner, god_type, agent_id, **kwargs)` — constructs `NPCAgent(agent_id=..., autonomous=True, mode="minecraft", god_type=god_type, **kwargs)` directly. Accepts `spawner` as its first parameter and never references it anywhere in its body — confirmed by reading the full four-line function.

## Problems (faced by traditional AI systems / LLMs)

Resuming persistent state across a process restart — reconnecting to whatever was already running rather than starting cold — is a general problem for any long-lived multi-agent system, distinct from the problem of creating an agent in the first place. It requires the restart-time discovery step and the creation-time step to agree on where things live and how they're named; if the two drift independently (as they clearly have here), discovery silently finds nothing rather than failing loudly, because an empty scan result and "nothing was ever packaged" look identical from the caller's side.

## Solutions

The parts of this file that don't depend on the naming match are genuinely solid: the event-based `wait_for_all_connections()` (a real race against a timeout, not a poll loop), the Minecraft-availability check with automatic frontend-only fallback, and `return_exceptions=True` in the gather so one agent's launch failure doesn't take down the others. None of that reaches the naming problem itself, which doesn't get solved in this file — the fix, whenever it happens, is a one-line change to the glob pattern (or, better, deriving both sides from one shared constant instead of two independently-written literals, given this is now confirmed to be exactly how they drifted apart the first time).

## Files Required

- [[config-two-copies]] — `py_backend/config.py` (`Config`).
- `py_backend/utils/mc_uuid.py` — `AgentNameManager` (lazy, two call sites; not yet ingested).
- `ai_core/agent.py` — `NPCAgent` (lazy, two call sites; not yet given its own page despite being one of the most-referenced files in this whole ingest — [[agent]] is that page).
- `py_backend/chat_launcher.py` — `start_chat_interface` (lazy; not yet ingested — next in queue).

## Files Used In

- [[main]] — the sole caller of `integrate_with_backend()`, at module level; also calls `scan_agents_folder()` and `launch_all_agents()` again inside `lifespan()`'s startup block, and `mark_connected()` from `/api/player_event`. Two small additions made to that page from this ingest (the four `/api/autoconnect/*` routes, and the `mark_connected()` call) — nothing on that page was wrong, just incomplete, since this file hadn't been read yet when it was written.
- [[auto_packager]] — `EnhancedAgentSpawner` is what `agent_manager.spawner` actually is, passed into `launch_all_agents(spawner)` — but, per the finding above, never actually called through that reference anywhere in this file.