---
type: reference
status: ingested
---
# Known Issues

💡 **What this is**: a synthesis of `Problems.md`, `AUDIT_REPORT.md`, and `FIXES_APPLIED.md` — three separate running documents in the repo root that overlap in places. This page exists so a future session doesn't have to cross-reference all three by hand. (Per Devlord: the `docs/` folder is separately outdated/broken and excluded from this wiki entirely — not the same thing as these three files, which are current.)

## Resolved: agent packaging & Minecraft launch (Problems.md, FIXES_APPLIED.md)

Two related bugs, both fixed:

1. **Packaging race condition** — `wait_for_connection()` was being called before `UltimMC` (the bundled Minecraft instance launcher — see [[architecture-overview]] → Packaging) was actually packaged and started, causing timeouts. Fixed by checking UltimMC availability first: if present, wait for the Minecraft connection (up to 120s); if not, log clearly that manual setup is needed and don't block agent startup on it.
2. **Wrong UltimMC executable** — originally attributed to `_create_portable_package()` in `packager.py` not copying the UltimMC folder into the portable package, even though `_setup_ultimmc()` had already prepared it. **Correction, confirmed against current [[wiki/codebase/files/packager|packager.py]]**: in the code as it stands today, `_create_portable_package()` doesn't copy UltimMC, mods, or frontend at all — it only checks expected files exist and writes `README.md`. The actual copy happens earlier, directly in `_setup_ultimmc()` (called from `package_agent()` before `_create_portable_package()` ever runs), into `agent_dir/UltimMC` — and since `agent_dir` and the final package directory are the same path, the net effect described (UltimMC present in the finished package) does still hold, just via a different method than originally attributed. Likely a refactor since this bug was first written up, not a re-introduction of the original bug.

**Preferred launch pattern** (from `Problems.md`, still relevant if touching `packager.py`):

```python
def try_ultimmc(server_addr, agent_name):
    ult = AGENT_DIR / "UltimMC" / "bin" / "UltimMC"
    if not ult.exists():
        return False
    subprocess.Popen([
        str(ult), "-d", str(AGENT_DIR / "UltimMC"),
        "-l", agent_name, "-s", server_addr, "-a", agent_name, "-o", "-n", agent_name,
    ], cwd=str(AGENT_DIR))
```

Using `-s <server> -l <instance> -o <agent_name>` handles account creation and mod registration automatically — this is the "preferred method," called out explicitly as such in the source doc. **One flag has drifted from this historical snippet, confirmed against the actual generated launcher today**: current code passes `-l AGENT_ID` (the stable internal identifier), not `-l agent_name` (the mutable display name) as shown above — sensible, since an instance-folder label shouldn't change just because a display name is edited later. The rest of the pattern matches exactly.

## New, not yet fixed: startup auto-launch never finds anything (found 2026-07-05, [[auto_connect_system]])

A third bug in the same general area as the two above, but this one is currently open, not resolved. `main.py`'s "auto-join all packaged agents on startup" feature — meant to relaunch every previously-packaged agent when the server restarts — cannot fire for any agent, ever, under the current code. [[auto_connect_system]]'s `scan_agents_folder()` globs `DW_*`-prefixed directories; [[wiki/codebase/files/packager|packager.py]]'s actual, current agent-directory naming is plain `{agent_id}`, no prefix (confirmed by that file's own in-line "no `DW_` prefix" comments at the real construction site, and by a matching comment in `main.py`). The scan runs twice on every startup and returns empty both times regardless of how many agents actually exist. Full evidence chain and a second, independent issue in the same code path (the `spawner` parameter threaded through `launch_all_agents()` is never actually called) on [[auto_connect_system]]. Notably, [[wiki/codebase/files/packager|packager.py]]'s own top-of-file docstring still describes the old `DW_{agent_id}` convention this glob was evidently written against — the naming changed in one place and not the other three, rather than this being an isolated typo.

## New, not yet fixed: two offline-UUID implementations that don't agree (found 2026-07-05, [[mc_uuid]] / [[minecraft_launcher]])

Also currently open. [[mc_uuid]]'s `get_minecraft_uuid(username)` and [[wiki/codebase/files/minecraft_launcher|minecraft_launcher.py]]'s `UltimMCLauncher._offline_uuid(username)` both claim to compute the same thing — Minecraft's offline-mode account UUID — and don't. [[mc_uuid]] hashes exactly `"OfflinePlayer:{username}"`'s UTF-8 bytes with MD5, matching Java's `UUID.nameUUIDFromBytes()` exactly (well-documented JDK behavior: no namespace prepending, despite the superficial "UUID v3" association). [[minecraft_launcher]] calls Python's `uuid.uuid3(NIL_UUID, "OfflinePlayer:{username}")`, which — per RFC 4122, which Python's stdlib correctly implements — prepends the namespace UUID's own bytes before hashing. Different input bytes, different MD5, different UUID, same username. This is verified directly by comparing what each function actually hashes, not inferred from the two having similar-sounding names or from any external tool's behavior — both implementations are in this codebase, in Python, and the JDK method they're each trying to match is unambiguous, standard, documented behavior. Not personally tested against a running Minecraft client, so the exact practical failure mode (skin/identity mismatch, a duplicate-seeming account, something else) isn't confirmed — but the mismatch between the two functions' outputs for the same input is arithmetic, not speculation.

## Audit findings (AUDIT_REPORT.md, dated May 21 2026)

22 issues found across Gradle/Java, Python, config, and docs. The one **critical** item:

- **Duplicate `Config` classes** — `py_backend/config.py` (canonical, imported by 8 files: main.py, packager.py, auto_connect_system.py, minecraft_launcher.py, auto_packager.py, chat_launcher.py, agent_spawner.py) and `py_backend/ai_core/config.py` (near-identical duplicate, imported only by `ai_core/__init__.py`). Risk: configuration drift between the two over time. Fix identified but not confirmed applied as of this ingest: delete the `ai_core` copy, repoint `ai_core/__init__.py` at the canonical one.

Also flagged: 3 duplicate `requirements.txt` entries (websockets was one), 2 Gradle inconsistencies between the client/server mod modules, 2 duplicate READMEs across the Java modules, 3 backward-compatibility wrapper/shim files, and 6 smaller structural issues. Not individually itemized here — see `AUDIT_REPORT.md` directly if working on any of these areas, since the full report has file:line references this page doesn't reproduce.

## A discrepancy this wiki itself surfaced

Not from any of the three source docs, but worth recording here since it c "roughly one-minute" BrainCapsule save cadence as a build target; the actual current code (`run_standalone_agent`'s main loop) saves every 300 seconds. See [[agent-runtime]] for the detail. Not a bug in either doc individually — the design wiki is describing a target for hardware that doesn't exist yet — but the numbers don't match today, and it's the kind of thing a lint pass across both wikis should keep catching.

## Electron frontend — small findings from its first ingest pass (2026-07-05)

Two unused callback props, same pattern in both files: `MentalMatrixSimulator.jsx` accepts `onSimulationEvent` and `WorldModelVisualizer.jsx` accepts `onPredictionRequest`, and neither ever actually calls the prop it's given, despite both parent components passing in a real callback expecting it to fire. Practical effect: `MentalMatrixModal`'s "Dump Buffer" export will always be empty, and nothing ever requests a prediction. See [[electron-frontend]].

`ControllerSafety.jsx` hardcodes `BACKEND_URL = "http://127.0.0.1:11400"` and `agent_id: "demo"` in three separate places, instead of using the dynamic values `App.jsx` derives from the page's own URL and `/status` — Controller Mode will silently talk to the wrong backend for any agent not running on port 11400.

## Deprecated, not a "problem" — just don't re-document as current

`py_backend/oracle_stuff/` (`create_oracle.py`, `teach_oracle.py`, `create_custom_texts.py`) — confirmed by Devlord as old and unused. See [[oracle-two-systems]] for the full context. Recorded here too since "old scripts still sitting in the repo" is exactly the kind of thing a future ingest pass might otherwise stumble on and assume is live.