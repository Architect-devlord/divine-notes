---
type: file
status: ingested
---
# main.py

💡 **Role**: the central multi-agent management server — the one process that spawns, tracks, packages, and stops every other agent process, serves a full browser-based admin GUI (agent list, personality/memory editors, live log stream, all self-contained in one embedded HTML/CSS/JS string), and is the actual HTTP surface the Java server mod calls into for genesis, breeding, player-join, and proximity-chat events. No dedicated topic-level page exists for this file specifically — [[architecture-overview]] covers the surrounding four-piece architecture and the port scheme this file implements, but not its own internals.

**Resolves the open question flagged on [[chat_system]] and in the last `log.md` entry.** Read directly: `main.py` never imports, references, or routes through `chat_system.py` — zero occurrences of that name anywhere in this file. Nor does `agent.py`, nor `auto_connect_system.py` (the only other module `main.py` itself wires into the FastAPI app via `integrate_with_backend()`). The only live reference to `chat_system.py` anywhere in the codebase is one narrow, `try/except`-guarded, best-effort call from [[cognitive_loop]]'s `_execute_file_processing()` — and that call turns out to be functionally inert too. `chat_system.py` is not the live routing path, and "superseded legacy layer" undersells how disconnected it is — nothing found this pass shows it was ever reachable from a running system in this commit, not just that something newer replaced it. Full trace below, under **The chat_system.py question, resolved**.

## Imports

External: `asyncio`, `json`, `logging`, `threading`, `os`, `re`, `subprocess`, `sys`, `time`, `contextlib.asynccontextmanager`, `pathlib.Path`, `typing` (`Any`/`Dict`/`List`/`Optional`), `psutil`, `uvicorn`, `argparse`, `fastapi` (`FastAPI`, `HTTPException`, `Request`, `WebSocket`, `WebSocketDisconnect`), `fastapi.middleware.cors.CORSMiddleware`, `fastapi.responses` (`HTMLResponse`, `JSONResponse` — the latter imported, never used). Two more are dead on arrival: `signal` and `uuid` are both imported at module level and never referenced anywhere in the file — no `signal.signal(...)` handler is registered (shutdown goes through FastAPI's `lifespan` instead) and no `uuid.uuid4()`-style call exists (agent UUIDs come from `get_minecraft_uuid()`, below).

Internal, eager: `py_backend.config.Config` (see [[config-two-copies]] — specifically the `py_backend.*`-prefixed copy, matching that page's "repo-root-invoked" convention), `ai_core.agent_spawner.AgentSpawner` (imported, never used — see Problems/Solutions), `ai_core.personality.assign_npc_gender`/`assign_god_gender` (also imported, also never called), `py_backend.auto_packager.EnhancedAgentSpawner` (not yet ingested at file level — the class this file actually instantiates), `py_backend.auto_connect_system.integrate_with_backend` (not yet ingested — called once at import time to attach additional routes to `app`), `py_backend.utils.mc_uuid` (`get_minecraft_uuid`, `AgentNameManager` — not yet ingested), `py_backend.utils.agents_json_manager.get_manager` (aliased `get_agents_manager` — not yet ingested), `py_backend.minecraft_launcher` (`UltimMCLauncher`, `MultiAgentLauncher` — not yet ingested).

Internal, guarded: `ai_core.logger_setup.initialize_logging` — wrapped in `try/except ImportError`, falling back to plain `logging.basicConfig()`. See [[logger_setup]].

Internal, lazy (imported inside a function body, at call time rather than module load): `ai_core.brain_capsule.BrainCapsule` (see [[brain_capsule]] — used by every brain-editor route and the breeding-fallback path), `py_backend.breeding_system.BreedingSystem` (see [[breeding_system]] — imported inside `lifespan()`), `ai_core.agent.NPCAgent` (see [[agent]] — imported inside `start_chat_interface()`).

## Module-level state

- `name_manager = AgentNameManager()` — display-name ↔ port resolution and random-name generation, used throughout.
- `_gui_log_handler = _GuiLogHandler()` — attached directly to the root logger (`logging.getLogger().addHandler(...)`), so every log record anywhere in the process (not just this file's own `log`) is captured and offered to GUI WebSockets.
- `server_integration = _ServerIntegrationShim()` — one instance of the backward-compat class below.
- `agent_manager = AgentProcessManager()` — the single central instance nearly every route in this file operates through.
- `_gui_sockets: List[WebSocket] = []` — tracked (appended on connect, removed on disconnect in `ws_gui()`) but never iterated or read anywhere else in the file. `_gui_log_handler`'s own separate internal `_sockets` list is what actually receives pushed messages; this one is dead weight, confirmed by a direct grep for every use of the name.
- `startup_time` — set once inside `lifespan()`, read by `/health`/`/health/detailed` for uptime.
- `GUI_HTML` — a single ~830-line self-contained HTML/CSS/JS string constant; see the GUI section below.

## Classes

### `_GuiLogHandler(logging.Handler)`

A `logging.Handler` subclass that turns every log record in the process into a GUI push. `MAX_BUFFER = 500`.

- `__init__()` — sets up an internal ring buffer, a `(WebSocket, event_loop)` list, and a `threading.Lock` (this handler's `emit()` can be called from any thread, since Python logging is thread-safe by default).
- `emit(record)` — formats the record, appends to the capped buffer, then for each subscribed socket schedules delivery via `loop.call_soon_threadsafe(loop.create_task, ws.send_json(...))` — the loop used is the one captured at subscribe time, not a fresh `get_event_loop()` call, so delivery targets the correct loop even when `emit()` fires from a non-async thread.
- `subscribe(ws, loop)` / `unsubscribe(ws)` — lock-guarded list mutation.
- `get_buffer()` — returns a copy of the current buffer, used to replay recent logs to a freshly-connected GUI socket.

### `_ServerIntegrationShim`

A thin backward-compatibility wrapper so existing code importing a `server_integration` object keeps working after `MinecraftServerIntegration` was collapsed (per the FIX comment directly above it — the class had gone vestigial once `usercache.json` handling moved elsewhere). Delegates everything to the two module-level functions below it.

- `server_folder` _(property)_ — `Config.SERVER_FOLDER` if it exists on disk, else `None`.
- `list_registered_agents()` — delegates to the module-level function of the same name.
- `register_agent(*args, **kwargs)` — delegates to the module-level function of the same name.

### `AgentProcessManager`

The actual orchestrator. One instance (`agent_manager`) owns every running agent subprocess, every launched Minecraft client process, and the packaging pipeline that connects them.

- `__init__()` — three dicts (`agent_processes`, `agent_info`, `minecraft_processes`), plus a `MultiAgentLauncher` (`ultimmc_launcher`), a `UltimMCLauncher` configured from `Config.CLIENT_JAR`/`Config.MOD_JAR` (`source_launcher`), and an `EnhancedAgentSpawner` configured for auto-packaging (`spawner`).
- `_generate_agent_uuid(agent_id, agent_type, custom_name)` — resolves a display name via `name_manager`, then calls `get_minecraft_uuid(username)`.
- `start_agent_process(agent_id, mode, server_addr, load_brain, additional_args, agent_type, custom_name, memory_mb, gender)` — the core spawn path. If a packaged executable already exists at `npc_applications/<id>/<id>` (no `DW_` prefix — matches `packager.py`'s own `exe_name` convention), runs that directly; otherwise resolves a username/UUID, registers the agent via `server_integration.register_agent(...)` (now passing `gender` as a real parameter — see Problems/Solutions, a dead-parameter bug fixed here), resolves stable TCP/WS ports from `agents.json` (registering a new name if one isn't found yet), and launches `ai_core/agent.py` as a subprocess with `--agent-id`/`--mode`/`--port`/`--tcp-port`/`--log-level`/`--brain-save-path` and optionally `--custom-name`/`--load-brain`/whatever `additional_args` the caller supplied. Schedules two background tasks: `_monitor_logs()` and `_auto_package_agent()`. Deliberately does **not** call UltimMC setup/launch itself — see the next method for why.
- `_auto_package_agent(agent_id, agent_type, custom_name, server_addr, memory_mb)` _(async)_ — polls (up to 3000s) for the agent's `brain.pcap` to appear, then packages it via `self.spawner.package_agent(...)`. Only after packaging succeeds does it call `ultimmc_launcher.setup_agent()` then `launch_agent()` — the packaged UltimMC copy has to exist first, so MC setup/launch is chained after packaging rather than done eagerly in `start_agent_process()`. If `mode != "minecraft"` or the agent is already in `minecraft_processes`, returns early without touching UltimMC at all.
- `_monitor_logs(agent_id, process)` _(async)_ — reads the subprocess's merged stdout/stderr line-by-line (`stderr=subprocess.STDOUT`, specifically to avoid a 64KB pipe-blockage deadlock) and re-logs each line prefixed with the agent ID; on exit, marks the agent `"stopped"` and records its exit code.
- `stop_agent_process(agent_id)` — terminates the Minecraft process (10s grace, then kill) if one exists, then the agent subprocess itself (30s grace — brain save is a ~90MB `torch.save`, so this is generous on purpose — then kill).
- `list_running_agents()` / `get_agent_status(agent_id)` — plain dict reads.
- `cleanup_all()` — stops every Minecraft process, then every agent process, then calls `self.spawner.cleanup_all()` if it exists.

## Module-level FastAPI app & routes

`app = FastAPI(..., lifespan=lifespan)`, CORS wide open (`allow_origins=["*"]`, `allow_credentials=False` — has to be `False` alongside a wildcard origin, noted in-line as FIX M-01). Immediately after construction: `agent_manager = AgentProcessManager()` then `integrate_with_backend(app, agent_manager)` — this is `main.py`'s one call into `auto_connect_system.py`, not yet ingested at file level; not yet confirmed what routes it adds beyond what's catalogued here.

**GUI & its WebSocket**

- `GET /gui` — returns `GUI_HTML` directly as an `HTMLResponse`.
- `WS /ws/gui` — accepts, subscribes to `_gui_log_handler`, starts a 20s keepalive ping task, replays the buffered log history, pushes the current agent list (`_push_agents()`), then loops on incoming messages handling only `{"type": "hello"}` (re-pushes the agent list) and silently ignoring everything else. This socket is purely GUI-management traffic (logs, agent list) — it carries no chat content of any kind, confirmed by a direct search: the string `"chat"` does not appear anywhere inside `GUI_HTML`.

**Brain editor (GUI-only)**

- `GET /api/agents/{agent_id}/brain` — loads a `BrainCapsule` off disk and returns personality/memories/gender/agent_type/metadata.
- `POST /api/agents/{agent_id}/brain/personality` — overwrites trait values (clamped to `[-1, 1]`) in the capsule and saves.
- `POST /api/agents/{agent_id}/brain/memories` — replaces `memory_snapshot` wholesale and saves.
- `POST /api/agents/{agent_id}/brain/config` — updates `agent_type`/`gender` in the capsule, and `server_addr`/`backend_port` in the live `agent_info` dict if the agent is currently running; explicitly notes changes take effect "on next agent restart," not live.

**Server & registration**

- `POST /api/server/configure`, `GET /api/server/status`, `GET /api/server/agents` — thin reads over `Config.SERVER_FOLDER` and `list_registered_agents()`/`server_integration.list_registered_agents()`.

**Spawn & agent lifecycle**

- `POST /api/agents/spawn_single` — the GUI's "spawn NPC" action and the Java mod's own single-NPC spawn path. Resolves a random name/gender/default-personality if not supplied, de-duplicates the agent ID against existing `npc_applications/` folders, and calls `start_agent_process()`.
- `POST /api/agents/start` — a more generic version of the above, mode defaults to `"autonomous"` rather than `"minecraft"`, accepts a raw `args` list passed straight through as `additional_args`.
- `POST /api/agents/{agent_id}/stop`, `GET /api/agents/{agent_id}/status`, `POST /api/agents/{agent_id}/package`, `POST /api/agents/{agent_id}/cleanup` — direct wrappers over the matching `AgentProcessManager` methods.
- `GET /api/agents/list` — merges currently-running agents with any `brain.pcap` found under `Config.BRAINS_DIR` that isn't currently running (`available_brains`), so the GUI can show stopped-but-resumable agents too.

**World events from the Java mod**

- `POST /api/player_event` — `"connected"` auto-starts the agent's backend process if it isn't already running (god agents get `--gender dual`), returns any stored `spawn_pos` so Java can teleport the agent on join; `"disconnected"` is acknowledged only, no process action taken.
- `POST /api/breeding/event` — routes through `app.state.breeding_system.initiate_breeding()` (see [[breeding_system]]) when available, computing real pregnancy/child-gender/due-time data; falls back to a direct-spawn offspring creation (blended-personality-plus-jitter, no pregnancy period) if `BreedingSystem` isn't wired into `app.state` yet, logging a warning when that happens. Comment marks this FIX B-04.
- `POST /api/agents/chat_heard` — **calls a method that does not exist.** Its own docstring is explicit about intent: NPC agents get overheard proximity chat over their persistent `/ws/agent` WebSocket, but god agents (per this docstring: _"run a separate LLM brain (`LLMOracleBrain`) and are not connected via a persistent WebSocket"_ — a detail not previously surfaced anywhere in this ingest; not chased down this pass, flagged below) need an HTTP fallback instead. The handler is supposed to push the overheard message into `agent_manager.get_chat_queue(hearer_id)` — but `get_chat_queue` is never defined anywhere in this file, on `AgentProcessManager`, or anywhere else in the repository (confirmed by a repo-wide search). Every call to this route raises `AttributeError` at that line, which FastAPI turns into an unhandled 500. There is currently no working path for a god agent to receive overheard proximity chat via this endpoint.

**Genesis, divine reset, gods**

- `POST /api/genesis/spawn` — spawns the two named "Adam"/"Eve" ancestor NPCs (hardcoded starter personalities) at each of the given `spawn_positions`, tagging each with `--genesis-ancestor true`.
- `POST /api/divine_reset` — stops every given agent's process and deletes its brain file. No confirmation step server-side (the GUI's own `confirm()` dialog is the only guard).
- `POST /api/agents/clear_memories` — wipes `memory_snapshot`/`language_state` on the given agents' brain capsules, respecting an `exceptions` list.
- `POST /api/gods/spawn` — the real god-spawn path (this is what `doSpawnGod()` in the GUI calls). Auto-selects a god type if none given, resolves a display name, and calls `start_agent_process()` with `--gender dual`, `--god-type <type>`, and the god's hardcoded `persona_traits`. A FIX comment here documents two now-fixed argument bugs found by diffing against the working NPC-spawn call site: `--memory-mb` was never a real `agent.py` flag (agent.py's `argparse` uses the strict `parse_args()`, so this used to crash the subprocess immediately on every god spawn), and `--god-type` was never being passed at all despite `agent.py` requiring it with no default — meaning any spawn that got past the first bug would still start with `god_type=None`.
- `POST /api/gods/ability` — **a stub.** Reads `agent_id`/`ability` from the request body and echoes them back with `status: "success"`; does not call `agent.use_god_ability()`, does not forward anything to the agent process or the Java mod. Unrelated to how abilities actually fire in practice — see [[god-abilities]] for the real mechanism (`use_god_ability()` → `act_god()`'s return value → `communication_protocol.py`'s `ActionFrame` builder).
- `POST /api/gods/transform` — **also a stub**, same shape (echoes `agent_id`/`target_mob`, does nothing else).

**Health & root**

- `GET /health` — status, running-agent count, uptime.
- `GET /health/detailed` — adds `psutil`-sourced CPU/memory.
- `GET /` — service banner pointing at `/gui`, plus the current running-agent list.

## Functions (module-level, outside the app/class)

- `_extract_spawn_pos(args)` — pulls `--spawn-x`/`-y`/`-z` values out of an `additional_args` list; returns `{}` unless all three are present.
- `sanitize_agent_id(name)` — lowercases, replaces spaces with underscores, strips anything outside `[a-z0-9_]`.
- `list_registered_agents()` — reads `agents.json` via `get_agents_manager()` and returns every male NPC / female NPC / god as `{"agent_id": ..., "type": ...}`.
- `register_agent(agent_id, agent_uuid, agent_type, custom_name, gender)` — writes into `agents.json` only (usercache files are the MC server's own concern, not this file's). For a god, registers directly under its god type. For an NPC, resolves gender in order: caller-supplied → already-registered-as-male → already-registered-as-female → default `"male"` with a logged warning. The in-line FIX comments describe this resolution order as already correct (this is FIX 2's actual implementation) — see Problems/Solutions for the _other_, still-hardcoded gender-guessing logic elsewhere in this file that this fix didn't reach.
- `lifespan(app)` _(async context manager)_ — startup: logs a banner, wires `BreedingSystem` into `app.state` (cross-wiring the spawner's own `_breeding_system` attribute so `EnhancedAgentSpawner._post_spawn()` can auto-attach new agents to it too), starts a 10-second `_breeding_tick_loop()` background task, then auto-launches every already-packaged agent found by `auto_connect`'s `scan_agents_folder()`. Shutdown: calls `agent_manager.cleanup_all()`.
- `start_chat_interface(agent_id, brain_path, config)` — headed `"chat_launcher helper (used when main.py is imported as a module)"`; not called anywhere inside this file itself. Constructs an `NPCAgent` directly (see [[agent]]), loads its brain if the path exists, launches **this same file** as a subprocess (`uvicorn py_backend.main:app` on a configured port), then either serves the built Electron frontend (`npx serve`) or runs its dev server (`npm run dev`) if `Config.FRONTEND_DIR` exists, or just prints the backend/GUI URLs if it doesn't. Presumed caller is `chat_launcher.py` (not yet read — next in the ingest queue), based on the comment; not yet confirmed directly.
- `if __name__ == "__main__":` entry point — parses `--port`/`--host`/`--reload`/`--cli`/`--gui` (mutually exclusive), prompts interactively for CLI-vs-GUI if neither flag is given, opens a browser to `/gui` on a background thread if GUI mode, then calls `uvicorn.run("main:app", ...)`.

## The chat_system.py question, resolved

Every claim below was checked directly against the code, not inferred from comments or docstrings alone.

**`main.py` itself**: zero references to `chat_system`, `cognitive_loop`, or `communication_protocol` anywhere in the file — confirmed by direct search, not absence-of-memory. The four functions `chat_system.py`'s own module docstring describes as _"Public API (used by main.py / FastAPI endpoints)"_ — `handle_frontend_chat`, `handle_frontend_file_upload`, `handle_minecraft_frame`, `start_agent_autonomous_speech` — have no call sites anywhere in the repository outside their own definitions.

**`agent.py`**: also zero references to `chat_system`. Its own `/ws` route (plain JSON messages, `{"type": "chat"}`) calls `global_agent.process_chat(message)` directly — the same method its `/chat` HTTP route uses (per [[agent]]) — with no `hasattr()` guards of any kind. This, not `chat_system.py`, is the actual live inbound-chat path for NPC agents.

**`auto_connect_system.py`** (the one other module `main.py` wires in): zero references to `chat_system` or `.brain`.

**The one surviving reference, anywhere**: [[cognitive_loop]]'s `_execute_file_processing()` has `try: from chat_system import chat_system; await chat_system.send_message(...) except Exception: pass` — a single, narrow, best-effort push of a "file processed" summary to GUI/game chat after `agent.brain.learn_from_file()` completes. Three independent reasons this call is functionally inert even when it doesn't raise:

1. **Packaging.** `py_backend/config.py`'s `AGENT_EXCLUDE_MODULES` explicitly excludes `py_backend.chat_system` from the packaged agent build (comment: _"server-side router, not agent"_) — confirmed identical in `ai_core/config.py`'s copy too, one of the few sections where the two copies agree byte-for-byte (contrast [[config-two-copies]]'s own `agent_spawner` mismatch finding). `cognitive_loop.py` itself **is** bundled into every packaged agent. So in a packaged, standalone-exe agent, this import fails and the `except Exception: pass` silently swallows it.
2. **Even unpackaged, the import target is process-local.** `main.py` spawns each agent as its own OS subprocess (`AgentProcessManager.start_agent_process()`, confirmed above). `chat_system.py`'s `chat_system = UnifiedChatSystem()` singleton is constructed at import time — so even in a dev/direct-run context where the import succeeds, each agent subprocess gets its own separate, freshly-constructed instance, never the same object any hypothetical central caller would use.
3. **Nothing ever registers a connection with it.** `UnifiedChatSystem.register_gui`/`register_game`/`register_agent` have no call sites anywhere outside `chat_system.py` itself — confirmed by repo-wide search. Per [[chat_system]]'s own already-documented behavior, `send_message()` queues regardless of whether a target connection exists; with nothing ever having registered one, the message goes into an internal capped queue nothing else reads.

**A genuine new nuance, not fully chased down**: `/api/agents/chat_heard`'s docstring (this file) states god agents run a separate `LLMOracleBrain`, distinct from the `BrainCore`/`LanguageIntelligence` combination confirmed everywhere else in this ingest. If true, `chat_system.py`'s `hasattr()` defensiveness — the thing that originally read as unexplained paranoia, since every NPC-side brain this ingest touched always implements `process_language_input`/`should_speak`/etc. — may have been reasonable defensive coding against **god** agents specifically, not paranoia about NPC agents at all. `LLMOracleBrain` itself hasn't been located or read this pass; it isn't `py_backend/oracle_stuff/` by name (that's confirmed deprecated — see [[oracle-two-systems]]) but its actual location and relationship to Oracle's already-documented two-systems split is unconfirmed. Worth a direct look, not assumed either way.

**A second, unrelated thread surfaced in passing**: `py_backend/utils/chat_backend.py` (queued, not yet read) defines its own module-level `handle_user_message(data)` — a naming coincidence with `chat_system.py`'s method of the same name, in a file whose name suggests it may be a third implementation of "handle inbound chat," possibly the actually-current one given its location alongside this file's other `utils.*` imports. Not chased down; flagged for when `utils/` comes up.

**Bottom line**: `chat_system.py` is not the live routing path, and "superseded legacy layer" undersells it — there's no evidence it was ever reachable from a running instance of this system as currently wired, not just that something newer took over. The real routing, for the parts of it that do work, is split three ways: `agent.py`'s own `/ws`/`/chat` for direct NPC chat, this file's `/api/player_event`/`/api/breeding/event` for world-state events, and this file's (currently broken) `/api/agents/chat_heard` for god proximity-chat specifically.

**Devlord's proposed direction for the fix, recorded here rather than left only in `log.md`**: route outbound agent speech through [[cognitive_loop]]'s own gated decision cycle rather than through anything resembling `chat_system.py`'s independent, ungated autonomous-speech loop — so the agent can choose not to speak, not just default to speaking whenever a threshold clears. Worth being precise about what already exists versus what's newly proposed, confirmed by direct read this pass: `cognitive_loop.py`'s `_decide()` already defaults to `{'type': 'none', ...}` and only overrides it when `should_process_file`/`should_speak`/`should_act`/`should_learn`/`should_browse_web` actually clears its own threshold — a "does nothing this cycle" outcome already exists structurally, and speech specifically is already gated behind `language_desire > 0.4` **and** a cooldown, not unconditional. What Devlord is actually asking for, then, is less "add a nothing feature" (one already exists) and more "make sure future chat-delivery work is wired through this existing gate rather than reintroducing an independent, ungated path" — exactly the shape `chat_system.py`'s own dormant autonomous-speech loop would have been, had it ever been reachable. Not implemented as a code change this pass — this is a documentation ingest, not a `divine-world-core` edit — but recorded here and in `log.md` as the agreed direction for whenever inbound/outbound chat delivery gets rebuilt.

## Problems (faced by traditional AI systems / LLMs)

Two distinct general problems, both concretely visible in this one file. First: **process orchestration for a multi-agent system** — spawning N independent long-running workers from a central coordinator, where each needs a stable identity (here: TCP/WS ports, a Minecraft UUID) that survives restarts without central bookkeeping getting out of sync with what's actually running. Second, and more visible in this file than anywhere else read this pass: **API-surface drift** — a route existing, being documented in its own docstring, and looking complete from a route list is not evidence it's wired to the subsystem it claims to control. `/api/gods/ability`, `/api/gods/transform`, and `/api/agents/chat_heard` all demonstrate this independently, in the same file, in the same god/chat-adjacent area — two are stubs that were seemingly never finished, one calls a method that plainly never existed. None of the three would show up in a read of the route list alone; each needed its handler body actually read.

## Solutions

The process-orchestration problem gets a genuinely solid answer here: subprocess-per-agent isolation (one crash doesn't cascade), stable ports/UUIDs persisted through `agents.json` rather than re-derived per launch, and packaging chained strictly after brain-file creation rather than raced against it. The API-surface-drift problem does **not** get a solution in this file — it's the honest opposite finding. Fixing it isn't a documentation-pass task (this is an ingest, not a code change), but the shape of a real fix is visible from the working examples already in this same file: `/api/gods/spawn`'s own FIX comment shows the pattern (diff a broken call site against a working sibling call site, here NPC-spawn vs. god-spawn, to find exactly where the argument list diverges) — the same technique would immediately show `/api/gods/ability`/`/api/gods/transform` never got a body past the stub, and that `get_chat_queue` needs either a real implementation on `AgentProcessManager` or the route needs to be rethought entirely.

## Files Required

- [[config-two-copies]] — `py_backend/config.py`, the `py_backend.*`-prefixed copy.
- [[agent_spawner]] — `ai_core/agent_spawner.py` (`AgentSpawner`), imported but unused; the actual class used is `EnhancedAgentSpawner`.
- [[personality]] — `ai_core/personality.py` (`assign_npc_gender`, `assign_god_gender`), also imported but unused.
- [[brain_capsule]] — `ai_core/brain_capsule.py` (`BrainCapsule`, lazy import), used throughout the brain-editor routes.
- [[breeding_system]] — `py_backend/breeding_system.py` (`BreedingSystem`, lazy import inside `lifespan()`).
- [[agent]] — `ai_core/agent.py`: not Python-imported, but launched as a subprocess (`sys.executable ai_core/agent.py --agent-id ... --mode ...`) by `start_agent_process()`, and directly constructed (`NPCAgent(agent_id)`) inside `start_chat_interface()`. [[agent]]'s own "Files Used In" already documents this relationship from its side.
- [[logger_setup]] — `ai_core/logger_setup.py` (`initialize_logging`, guarded import).
- `auto_packager.py` (`EnhancedAgentSpawner`), `auto_connect_system.py` (`integrate_with_backend`), `utils/mc_uuid.py` (`get_minecraft_uuid`, `AgentNameManager`), `utils/agents_json_manager.py` (`get_manager`), `minecraft_launcher.py` (`UltimMCLauncher`, `MultiAgentLauncher`) — none has its own file-level page yet; all five are explicitly queued next. See [[architecture-overview]] for narrative coverage of the packaging/launch pipeline in the meantime.

## Files Used In

- `chat_launcher.py` — presumed caller of `start_chat_interface()`, based on this file's own in-line comment; not yet confirmed, since `chat_launcher.py` hasn't been read yet.
- The Java server mod (`DivineWorld`) — calls `/api/player_event`, `/api/breeding/event`, `/api/agents/chat_heard`, and presumably the genesis/god-spawn routes, per [[architecture-overview]]'s already-documented `PythonBackendClient.java` → REST API direction.
- Itself, indirectly — `start_chat_interface()` launches `uvicorn py_backend.main:app` as a subprocess, i.e. this file can end up re-invoking itself as an ASGI target under a second code path.