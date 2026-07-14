---
type: file
status: ingested
---
# chat_launcher.py

💡 **Role**: `start_chat_interface(agent_id, brain_path, config)` — launches a single agent's frontend-only chat interface (no Minecraft connection required): load/create the agent, attach a browser, start `main.py` as a backend subprocess, serve the React frontend, open a browser tab, block until exit. Its own module docstring is unusually direct about its own history — a "Changes from original" list, naming this as a modified descendant of another implementation.

**Resolves what [[main]] and [[auto_connect_system]] both left as "presumed, not confirmed."** `main.py` has its own, separate `start_chat_interface()` (same name, same core shape) with an in-line comment calling itself a "chat_launcher helper" — implying this file would call it. It doesn't. This file has a fully independent implementation of the same function name, and [[auto_connect_system]]'s `_launch_frontend_only()` imports _this_ one (`from py_backend.chat_launcher import start_chat_interface`) — confirmed by direct read, not the other way around. A repo-wide search for every reference to `start_chat_interface` turns up exactly three places: this file's own definition, this file's own `__main__` block calling it, and `auto_connect_system.py` importing it from here. **`main.py`'s own `start_chat_interface()` has zero callers anywhere in the repository — not presumed dead, confirmed dead.** Corrected on [[main]] below.

This file's own docstring lists three real deltas from whatever it's a modified copy of, all confirmed present in the code: a brain-path fallback (tries `Config.BRAINS_DIR/<agent_id>/brain.pcap` if the given path is empty or missing, rather than just failing), a browser attached automatically via [[web_browser]]'s `add_web_browsing_to_agent()` with `allowed_websites` loaded straight from `agents.json`, and the browser tab opening automatically (`webbrowser.open()`) once the frontend's up — none of which `main.py`'s own version has, per that page's existing description. A fourth difference, not called out in the docstring but real: this file also resolves a default backend port from `agents.json`'s stored TCP port (`+ 10000`, the `WS_PORT_OFFSET` seen elsewhere in this ingest) if the caller's `config` dict doesn't specify one — `main.py`'s version takes whatever port it's given without this fallback.

## Imports

`logging`, `subprocess`, `sys`, `time`, `webbrowser`, `pathlib.Path`, `typing` (`Any`, `Dict`) — all stdlib. Everything internal is imported lazily, inside the function body, each independently guarded: `ai_core.agent.NPCAgent` (hard-fails, re-raises, if missing — this one's not optional), `py_backend.config.Config` (falls back to hardcoded defaults on `ImportError` rather than failing), [[web_browser]]'s `add_web_browsing_to_agent` (soft-fails — logs and continues without a browser), `py_backend.utils.mc_uuid.AgentNameManager` (imported twice, independently, once for `get_allowed_websites()` and once for `get_port_for_name()` — each call wrapped in its own `try/except`, so a failure in one doesn't affect the other).

## Function

- `start_chat_interface(agent_id, brain_path, config)` — the whole file, functionally. Resolves the brain path with the fallback described above; constructs `NPCAgent(agent_id)` and loads the brain if found, otherwise logs a warning and starts fresh (never raises for a missing brain — this is explicitly the frontend-only, no-Minecraft-required path, so a fresh agent is an acceptable outcome, not an error); attaches a browser with allowed sites from `agents.json`; resolves a backend port (`config["backend_port"]` → `agents.json` TCP port `+ 10000` → `Config.BASE_BACKEND_PORT`/`11400`); starts `python -m uvicorn py_backend.main:app` as a subprocess on that port; serves the frontend (built `dist/` via `npx serve` on `:8765`, or `npm run dev` on `:5173` if unbuilt, or just prints the backend/GUI URLs if the frontend directory doesn't exist at all — matching `main.py`'s own three-way fallback exactly); opens a browser tab to whichever URL ended up being the real one; blocks on the backend subprocess until it exits or `KeyboardInterrupt`, terminating cleanly on the latter.
- `if __name__ == "__main__":` — a genuine standalone CLI: `python chat_launcher.py <agent_id> [brain_path] [backend_port]`, with basic logging configured fresh (this file doesn't reuse [[logger_setup]]).

## Problems (faced by traditional AI systems / LLMs)

Serving a chat interface for one specific agent, without requiring the rest of the system (a Minecraft server, a full multi-agent manager) to be running, is a genuinely different deployment shape than the main server — but reimplementing that shape as a completely separate function, under the same name as an existing one elsewhere in the codebase, is its own hazard: two implementations that started from the same idea can drift, and nothing forces a reader (or a future editor) to notice they're not looking at the one they think they are.

## Solutions

This file doesn't solve the drift risk — it _is_ an instance of it, confirmed by direct read rather than assumed. What it does solve well is the actual deployment problem: every external dependency here (`Config`, `NPCAgent`, the browser, `agents.json` lookups) is either lazily imported or wrapped in its own narrow `try/except` with a sensible fallback, so this file can genuinely run in a stripped-down context — no Minecraft, possibly no `Config` module resolving cleanly, possibly no browser dependencies installed — and still produce a working chat interface with degraded-but-functional behavior rather than failing outright.

## Files Required

- `ai_core/agent.py` — `NPCAgent` (not optional; the one import in this file that re-raises on failure). See [[agent]].
- [[config-two-copies]] — `py_backend/config.py` (`Config`), guarded, with hardcoded fallbacks.
- [[web_browser]] — `add_web_browsing_to_agent` (guarded).
- `py_backend/utils/mc_uuid.py` — `AgentNameManager` (guarded, two independent call sites; not yet ingested).

## Files Used In

- [[auto_connect_system]] — `_launch_frontend_only()` imports and calls this file's `start_chat_interface()` directly; the actual, confirmed live path.
- **Not** `main.py` — that file has its own, separate, less-capable function of the same name, confirmed to have zero callers anywhere in the repository. See the correction on [[main]].