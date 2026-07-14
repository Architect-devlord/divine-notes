---
type: file
status: ingested
---
# validation.py

💡 **Role**: six Pydantic `BaseModel`s — `ChatRequest`, `FileUploadRequest`, `AgentSpawnRequest`, `ControllerActivationRequest`, `TaskSwitchRequest`, `PlayerEventRequest` — a proper, dedicated request-validation layer. The module docstring states its purpose plainly: "Prevents injection attacks and ensures data integrity." Every field shape here closely mirrors real route parameters documented elsewhere this ingest — `AgentSpawnRequest`'s `agent_id`/`god_type`/`custom_name`/`gender`/`server_addr`/`persona_traits` line up almost exactly with [[main]]'s `/api/gods/spawn`; `ControllerActivationRequest`'s `permissions`/`camera_index`/`mic_device_index` with [[dw_controller]]'s activation flow; `PlayerEventRequest`'s `agent_id`/`player_uuid`/`event` with [[main]]'s `/api/player_event`.

**None of the six models is used anywhere.** Confirmed by direct search across `main.py` and `ai_core/agent.py` — the two files that actually define every route these models appear to be built for — zero references to any of the six class names, in either file. The only mention of this module anywhere outside itself is a string literal inside `config.py`'s `AGENT_HIDDEN_IMPORTS` list (`'py_backend.utils.validation', # request validation`) — a PyInstaller packaging hint, not an import, and evidence someone expected this module to matter at some point rather than evidence it currently does. Every route this ingest has actually read parses its own request body manually (raw `Request.json()`/dict `.get()` calls, documented at length across [[main]] and [[agent]]) rather than through this file's `Field(...)`/`@field_validator` machinery. A fitting note to close the whole Python-side ingest on: the file explicitly built to prevent injection and validate input across exactly the request types this session spent the most time reading is itself never actually invoked.

## Imports

`re`, `pathlib.Path`, `typing` — stdlib. `pydantic` (`BaseModel`, `Field`, `field_validator`) — external.

## Classes

- **`ChatRequest`** — `message` (1–10,000 chars), `agent_id` (alphanumeric/underscore/hyphen pattern, default `"demo"`). `sanitize_message()` strips, drops null bytes, collapses 4+ consecutive newlines to 3, and strips every character below ASCII 32 except newline/tab. `validate_agent_id()` additionally rejects `..`/`/`/`\` (path-traversal characters) even though the field pattern would already exclude `/`/`\` — the length and traversal checks are the validator's actual added value over the pattern alone.
- **`FileUploadRequest`** — `filename` (max 255), `agent_id`, `filetype`. `safe_filename()` takes only `Path(v).name` (discarding any directory component outright — the real path-traversal defense, stronger than a pattern match), strips to word characters/whitespace/dot/hyphen, prevents a leading-dot hidden file, substitutes a fixed name for an empty/whitespace result, and truncates long names while preserving the extension. `validate_filetype()` falls back to `application/octet-stream` for anything outside a fixed allow-list of MIME prefixes rather than rejecting the request outright.
- **`AgentSpawnRequest`** — `agent_id` (1–50 chars, patterned), `agent_type` (default `"npc"`), optional `god_type`/`custom_name`/`gender`/`persona_traits`, `server_addr` (default `"127.0.0.1:25565"`). `validate_agent_type()` accepts `"npc"`, `"god"`, or `"god_<type>"` for the six known god types — the docstring is explicit this three-way form is for backwards compatibility. `validate_server()` regex-checks `host:port` shape and range-checks the port 1–65535. `validate_traits()` silently drops any key outside the eight known personality trait names and clamps every kept value to `[-1, 1]`, rather than rejecting unknown keys.
- **`ControllerActivationRequest`** — `agent_id`, `permissions` (list, filtered to the four known permission names), `acknowledged`, `camera_index`/`mic_device_index` (0–10), `resolution` (defaults `[640, 480]`, clamped to a sane range or reset to default if malformed), `fps` (1–60), `mic_sample_rate` (8,000–48,000).
- **`TaskSwitchRequest`** — one field, `task_id` (0–1000).
- **`PlayerEventRequest`** — `agent_id`, `player_uuid` (validated and lowercased against the canonical 8-4-4-4-12 hex UUID shape), `agent_type`, `event` (patterned to exactly `"connected"` or `"disconnected"`).

## Problems (faced by traditional AI systems / LLMs)

Not AI/ML-specific — this file's problem is the general one of untrusted request input reaching filesystem paths, subprocess-adjacent identifiers, and stored text: path traversal via filenames, injection via unsanitized text fields, and malformed structured data (bad ports, out-of-range indices, unrecognized enum-like strings) reaching code that assumes well-formed input.

## Solutions

Where this file's own validators are actually exercised (they aren't, in production, per the finding above), the approach is sound: `Path(v).name` for filename traversal is a stronger defense than pattern-matching against forbidden substrings would be, since it structurally can't retain a `../` regardless of encoding tricks; clamping and filtering (personality traits, resolution, permissions) rather than rejecting outright is a reasonable choice for fields where a partially-valid request is still useful. None of this reaches production, though — the actual solution in place, across every real route this ingest read, is ad hoc per-field checks written inline at each call site, which is a real, working, if less centralized and less rigorous, alternative solution to the same problem.

## Files Required

None — `pydantic` only.

## Files Used In

Nothing found. Confirmed by direct search across every file where these six models would plausibly be used — `main.py` and `ai_core/agent.py`, the two files defining every route these models are shaped for. The one reference anywhere outside this file itself is a PyInstaller hidden-import string in `config.py`, not a usage.

---

**This closes the Python-side per-file ingest.** Every file listed at the start of this batch — `main.py`, `web_browser.py`, `packager.py`, `auto_packager.py`, `auto_connect_system.py`, `chat_launcher.py`, `frontend_builder.py`, `minecraft_launcher.py`, and all six of `utils/` — now has its own page. The Java mod layer and the Electron frontend remain topic-level only, per [[index]]'s own tracking.