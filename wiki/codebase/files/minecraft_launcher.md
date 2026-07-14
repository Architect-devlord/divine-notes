---
type: file
status: ingested
---
# minecraft_launcher.py

💡 **Role**: two classes. `UltimMCLauncher` is the cross-platform automation layer for one UltimMC installation — find it, copy it, create accounts and instances, install mods, launch. `MultiAgentLauncher` sits on top, managing one `UltimMCLauncher` per agent (`self.launchers: Dict[str, UltimMCLauncher]`) and the running processes that come out of them. Confirmed as the real class this file's imports resolve to on both sides that reference it: [[main]]'s `AgentProcessManager` constructs both a `MultiAgentLauncher` (`ultimmc_launcher`) and a standalone `UltimMCLauncher` (`source_launcher`, the template UltimMC install everything else clones from), matching this file's own constructors exactly.

**A note on the `-l` flag in `launch_instance()`'s launch command, checked directly with the project owner rather than assumed either way**: it's confirmed to select the Minecraft version being launched, not an instance identifier or account name — left exactly as written. Worth flagging why this needed checking rather than reading obviously: upstream MultiMC (which UltimMC is a fork of) documents `-l`/`--launch <instanceID>` as an instance selector in its own public issue tracker, and this codebase's own generated `packager.py` launcher passes what looks like an agent-identifying value to a same-named `-l` flag elsewhere — two different-looking uses of the same flag name, in two different files, that turned out not to be the same question. Confirmed directly rather than inferred from either side.

## Imports

`json`, `logging`, `os`, `platform`, `shutil`, `subprocess`, `time`, `uuid` (aliased `_uuid_mod`), `pathlib.Path`, `typing` — stdlib. [[config-two-copies]] (`py_backend.config.Config`) and `py_backend.utils.mc_uuid` (`get_minecraft_uuid`, `AgentNameManager`, not yet ingested) — both at module level, both eager.

## Classes

### `UltimMCLauncher`

Class attributes `MINECRAFT_VERSION`/`FORGE_VERSION` are copied from `Config` at class-definition time.

- `__init__(ultimmc_path=None, client_jar_path=None, mod_jar_path=None, custom_ultimmc_dir=None)` — resolves the source UltimMC install, the DWClientBot and DivineWorld jars, and (if a `custom_ultimmc_dir` was given) locates the actual executable inside it immediately.
- `_find_ultimmc(explicit)` — the same five-tier priority order already documented on [[wiki/codebase/files/packager|packager.py]]'s own `_find_ultimmc()` (`agents.json` → explicit arg/`DW_ULTIMMC_PATH` env → project-relative → platform-specific candidates → `PATH`), independently reimplemented here rather than shared — two files, same discovery logic, written twice.
- `_exe_hints(base)` _(static)_ — per-platform candidate executable paths inside a given UltimMC root; reused by `_locate_executable()`, `copy_ultimmc_installation()`, and `MultiAgentLauncher.create_launcher_for_agent()`.
- `_find_jar(explicit, filename)` — checks the explicit path first, then globs four fixed candidate roots.
- `_locate_executable()` — sets `self.ultimmc_executable` from `_exe_hints()`, `chmod`-ing it executable on non-Windows if needed.
- `copy_ultimmc_installation(dest_dir)` — clones the discovered UltimMC install (correcting for being pointed at a `bin/`/`MacOS/` subfolder rather than the true root first) into `dest_dir`, re-locating the executable afterward and failing cleanly if none is found post-copy.
- `create_account(username, make_active=True, custom_uuid=None)` — reads or initializes `bin/accounts.json`, builds an offline-style account entry (a fresh `clientToken` each time, `entitlement` flags set true), deactivates every other account first if `make_active`, replaces an existing entry with the same username in place rather than duplicating it.
- `_offline_uuid(username)` _(static)_ — **confirmed to diverge from [[mc_uuid]]'s `get_minecraft_uuid()`, not just superficially similar**: this method uses `uuid.uuid3(NIL_UUID, f"OfflinePlayer:{username}")`, which prepends the namespace UUID's bytes before hashing per RFC 4122. [[mc_uuid]]'s version hashes only the name string, matching Java's actual `UUID.nameUUIDFromBytes()` behavior (which does not prepend a namespace) exactly, per that file's own docstring. The two produce different UUIDs for the same username — verified directly by comparing what each function actually hashes, not assumed from the similar names. Full account on [[mc_uuid]].
- `create_instance(instance_name, forge_install=True)` — writes `instance.cfg` and `mmc-pack.json` (LWJGL 3.3.1 + Minecraft `self.MINECRAFT_VERSION`, optionally + Forge `self.FORGE_VERSION`) for a new instance folder.
- `_update_instance_cfg(instance_name, jvm_props, memory_mb=2048)` — rewrites only the managed keys (`OverrideJavaArgs`, `JvmArgs`, `OverrideMemory`, `Max/MinMemAlloc`) in an existing `instance.cfg`, preserving every other line already in the file rather than overwriting it wholesale.
- `install_mods(instance_name)` — copies both mod jars into that instance's `.minecraft/mods/`.
- `launch_instance(instance_name, server_addr=None, profile_name=None, offline=True, offline_name=None, agent_id=None, backend_url=None, memory_mb=Config.CLIENT_MEMORY_MB, extra_jvm_args=None, headless=False)` — resolves and validates the executable (correcting its path if it's not sitting inside a `bin`/`MacOS` folder as expected), builds the launch command (`xvfb-run -a` prefix if `headless` and on Linux), resolves the account/offline-name flags, builds JVM args (memory, `-Ddw.agent.id`, `-Ddw.backend.url`/`-Ddw.backend.port` parsed from `backend_url`, `-Ddw.tcp.port` looked up from `agents.json` via `AgentNameManager`, `-Ddw.server`, plus any `extra_jvm_args`), pushes the `-D` args into that instance's `instance.cfg` via `_update_instance_cfg()` before launching, then starts the subprocess with `INST_JAVA` explicitly stripped from the environment (an in-line comment notes this is the Java executable path itself, not JVM args, so it shouldn't be inherited here).

### `MultiAgentLauncher`

- `__init__(base_dir=None)` — `base_dir` defaults to `Config.NPC_APPLICATIONS_DIR`; holds `launchers` (per-agent `UltimMCLauncher`) and `processes` (per-agent running `Popen`) dicts.
- `create_launcher_for_agent(agent_id, source_launcher=None)` — checks for an already-packaged UltimMC copy at `Config.NPC_APPLICATIONS_DIR/<agent_id>/UltimMC` first (constructing a bare `UltimMCLauncher` via `__new__`, bypassing `__init__`, and hand-setting its attributes directly rather than re-running discovery); if none exists, clones fresh from `source_launcher` into `self.base_dir/<agent_id>/UltimMC` instead.
- `setup_agent(agent_id, server_addr=Config.DEFAULT_SERVER, custom_uuid=None, agent_type="npc", custom_name=None, source_launcher=None)` — resolves a username (`custom_name`, or `DWGOD_{TYPE}_{id}` for gods, or `DW_{id}` for NPCs — a third naming convention, distinct from both [[wiki/codebase/files/packager|packager.py]]'s own Minecraft-name resolution and its own `agent_id`-based UltimMC instance naming), a UUID if not given, creates the account and an instance named `f"agent_{agent_id}"`, installs mods.
- `launch_agent(agent_id, server_addr, backend_url, memory_mb=Config.CLIENT_MEMORY_MB, extra_jvm_args=None, headless=False, agent_type="npc", custom_name=None)` — requires `setup_agent()` to have run first (fails cleanly with a clear log message if not), resolves the same username convention independently a second time, calls `launcher.launch_instance()` with `instance_name=f"agent_{agent_id}"`.
- `launch_multiple_agents(agent_configs, delay_between_launches=2.0, source_launcher=None, headless=False)` — runs `setup_agent()` (if not already set up) then `launch_agent()` per config dict, with a delay between each launch, tolerating individual failures without stopping the batch.
- `stop_agent(agent_id, timeout=10)` / `stop_all_agents()` / `get_running_agents()` — plain process lifecycle management, `get_running_agents()` also pruning dead entries as a side effect of checking.

**Worth noting plainly, not as a problem**: this file's own `create_instance`/`setup_agent` path names each agent's UltimMC instance `agent_{agent_id}` (e.g. `agent_eve`), while [[wiki/codebase/files/packager|packager.py]]'s separate `_setup_ultimmc()` names its own clone target bare `agent_id` (`eve`, no prefix). The two don't appear to read each other's instance folders anywhere found this pass — they look like genuinely independent setup paths (one for building a portable exe package, one for a live, non-packaged launch) rather than two implementations of the same thing that need to agree on a name.

## Problems (faced by traditional AI systems / LLMs)

Not an AI/ML-specific problem — this file's problem is general to running many independent, stateful, GUI-oriented client processes (one Minecraft client per agent) from a headless backend: each client needs its own account, its own instance directory, its own mods, and its own JVM configuration, without the setup for one agent leaking into or overwriting another's.

## Solutions

Per-agent isolation is handled by giving every agent its own `UltimMCLauncher` instance pointed at its own cloned UltimMC directory, rather than sharing one central install across agents. `_update_instance_cfg()`'s line-preserving rewrite (only touching the specific keys it manages) avoids clobbering any other instance settings a user or a previous run might have set. The reused, already-packaged-install fast path in `create_launcher_for_agent()` avoids re-cloning a full UltimMC installation for an agent that's already been packaged once.

## Files Required

- [[config-two-copies]] — `py_backend/config.py` (`Config`).
- `py_backend/utils/mc_uuid.py` — `get_minecraft_uuid`, `AgentNameManager` (not yet ingested).

## Files Used In

- [[main]] — `AgentProcessManager.__init__()` constructs both a `MultiAgentLauncher` (`ultimmc_launcher`) and a standalone `UltimMCLauncher` (`source_launcher`), matching this file's constructors; `_auto_package_agent()` calls `ultimmc_launcher.setup_agent()` then `launch_agent()` after packaging succeeds.