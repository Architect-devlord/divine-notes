---
type: file
status: ingested
---
# mc_uuid.py

💡 **Role**: two things, both genuinely canonical. `get_minecraft_uuid(username)` — the one correct implementation, confirmed below, of Minecraft's offline-mode UUID algorithm. `AgentNameManager` — the single source of truth for `agents.json` (name↔port registry, server address, `minecraft_path`, `allowed_websites`), read from and written to by nearly every other file in this ingest. The most-referenced still-unread dependency across this entire session, cited from `main.py` onward — confirms every prior citation was accurate: this really is what those pages said it was.

**A real, verifiable divergence, confirmed entirely within this codebase rather than depending on any external tool's undocumented behavior**: [[wiki/codebase/files/minecraft_launcher|minecraft_launcher.py]]'s own `UltimMCLauncher._offline_uuid()` computes offline UUIDs a different way than this file does, and the two would produce **different UUIDs for the same username**. This file: `hashlib.md5(f"OfflinePlayer:{username}".encode("utf-8")).digest()` — hashes exactly those bytes, nothing else, then sets version/variant bits. `minecraft_launcher.py`: `uuid.uuid3(UUID("00000000-...-0000"), f"OfflinePlayer:{username}")` — Python's stdlib `uuid3()` prepends the _namespace UUID's own 16 bytes_ to the name before hashing, per RFC 4122's name-based-UUID spec. Even an all-zero namespace still changes the MD5 input (16 extra zero bytes prepended), so the two calls hash different byte strings and produce different results. This file's own docstring states precisely which Java call it's matching — `UUID.nameUUIDFromBytes(("OfflinePlayer:" + name).getBytes(...))` — and that method is well-documented, standard JDK behavior: it MD5s the given bytes directly, with **no** namespace prepending, unlike RFC 4122's UUIDv3. This is actually a known point of confusion in Minecraft tooling generally — `UUID.nameUUIDFromBytes` looks like or gets casually called "UUID v3," but doesn't follow the namespace-prepending convention true RFC 4122 v3 (and Python's `uuid.uuid3`) requires. This file's implementation matches Java exactly; `minecraft_launcher.py`'s does not. Not personally tested against a running Minecraft client to confirm the practical consequence, but the mismatch itself is arithmetic, not an inference — the two functions provably hash different inputs. Noted on [[minecraft_launcher]] and added to [[known-issues]].

## Imports

`hashlib`, `json`, `logging`, `os`, `random`, `uuid`, `pathlib.Path`, `typing` — stdlib only.

## Module-level

- `PORT_START = 11401` — every registered name (NPC or god) gets a port starting here, incrementing globally across every category, not per-category.
- `get_minecraft_uuid(username)` — see above; the correct offline-UUID implementation.

## Class

### `AgentNameManager`

- `SPAWNABLE_GOD_TYPES` — the six god types (matches every other list of the same six found throughout this ingest, including [[god-abilities]]'s and [[action_format_sync]]'s).
- `_load_default_names()` _(classmethod)_ — three-tier priority for the starter name pool: `default_agents.json` beside this file, then beside `config.py`, then a hardcoded fallback dict (mythological/biblical names — Adam/Eve/Thor/Zeus and so on) identical in content to what the shipped JSON is expected to contain. Reloads from disk on every call rather than caching, specifically so hand-edits to `default_agents.json` are picked up without a restart.
- `_build_default_content()` / `_default_content()` _(classmethods)_ — converts the flat name lists into `{name: port}` dicts with sequentially-assigned ports starting at `PORT_START`; the second one caches the first's result on the class itself after one call.
- `__init__(config_path=None)` — resolves the real `agents.json` location (`_find_config_path()`: `~/Documents`, `~/Desktop`, then the two OneDrive equivalents, first one that exists wins, defaulting to `~/Documents` if none do), creates it with default content if missing.
- `get_server_info()` / `get_minecraft_path()` / `get_allowed_websites()` _(classmethods)_ — read-only, tolerant helpers (empty/default return rather than raising on any failure) confirmed as the exact functions [[main]], [[web_browser]] (via [[agent]]/[[chat_launcher]]), and [[packager]]/[[minecraft_launcher]] all already cited. `get_minecraft_path()` has its own three-tier fallback (agents.json field → beside `py_backend` → one level up), independent of, and slightly different from, [[wiki/codebase/files/packager|packager.py]]'s and [[wiki/codebase/files/minecraft_launcher|minecraft_launcher.py]]'s own separate UltimMC-discovery logic (a third, independent implementation of "find UltimMC," alongside those two already-documented ones — not chased down further, since this one is narrower in scope, agents.json-and-two-fixed-paths only, not the full platform-specific candidate list the other two have).
- `load_config()` / `save_config()` — plain read/write, both tolerant (return defaults, or `False`, rather than raising).
- `_next_port(config)` — scans every existing port across NPCs and gods, returns one past the highest found (not `PORT_START` plus a count — so a gap from a deleted agent doesn't get reused, ports only ever increase).
- `get_port(name)` / `get_port_for_name(name)` (identical, second is an alias) — linear scan across every category for a matching name.
- `resolve_display_name(agent_id, custom_name=None)` — despite the name, this always returns `agent_id` unless `custom_name` is given and non-empty; the `get_port(agent_id)` check in the middle doesn't change the return value on either branch. Reads as either an incomplete implementation or a vestigial check whose original purpose (perhaps an earlier version returned something different when a port was already assigned) no longer does anything.
- `get_all_ports()` — flat `{name: port}` across everything.
- `get_random_name(category, subcategory)` — falls back to the built-in default pool if the live config's pool for that category/subcategory is empty (e.g., every name in the starter pool already used and removed).
- `add_name(category, subcategory, name)` — the actual registration write path. Includes a legacy-format migration: if the target turns out to be a list (an old format) rather than a dict, converts it in place, assigning sequential ports, before proceeding — a real, still-live migration path, not just a comment about one.
- `resolve_npc_name()` / `resolve_god_name()` / `register_npc()` / `register_god()` / `unregister_npc()` / `unregister_god()` / `get_all_male_npcs()` / `get_all_female_npcs()` / `get_all_gods()` / `get_stats()` — the higher-level API most callers actually use; all built on `add_name()`/`load_config()`/`save_config()` underneath.

## Problems (faced by traditional AI systems / LLMs)

Not AI/ML-specific — this file's real problem is identity stability for a system with many independently-launched processes: every agent needs a name, a port, and (for Minecraft ones) a UUID that stay the same across restarts, without a central always-running process to coordinate them, since agents are launched as separate subprocesses that each need to resolve the same answer independently.

## Solutions

A single JSON file as the shared, persistent source of truth, with tolerant read/write (defaults on any failure rather than crashing a caller that just wants to know a port) is a plain, correct answer to the coordination problem — every one of the many files that cite this one across this whole ingest reads the same file, so independently-launched processes do converge on the same answers. The one place this file's own solution doesn't hold together is the offline-UUID divergence above — the exact kind of "two independent implementations of the same well-defined algorithm" risk a single shared module is supposed to prevent, present here because `minecraft_launcher.py` reimplemented it rather than importing this file's version.

## Files Required

None — stdlib only.

## Files Used In

Cited, now confirmed accurate, by nearly every file this session: [[main]], [[packager]], [[auto_packager]], [[auto_connect_system]], [[chat_launcher]], [[minecraft_launcher]], [[web_browser]] (indirectly, via [[agent]] and [[chat_launcher]]), [[agents_json_manager]] (a direct re-export of this file's `AgentNameManager`).