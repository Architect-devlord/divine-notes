---
type: reference
status: ingested
---

# Known Issues

💡 **What this is**: a synthesis of `Problems.md`, `AUDIT_REPORT.md`, and `FIXES_APPLIED.md` — three separate running documents in the repo root that overlap in places. This page exists so a future session doesn't have to cross-reference all three by hand. (Per Devlord: the `docs/` folder is separately outdated/broken and excluded from this wiki entirely — not the same thing as these three files, which are current.)

## Resolved: agent packaging & Minecraft launch (Problems.md, FIXES_APPLIED.md)

Two related bugs, both fixed:

1. **Packaging race condition** — `wait_for_connection()` was being called before `UltimMC` (the bundled Minecraft instance launcher — see [[architecture-overview]] → Packaging) was actually packaged and started, causing timeouts. Fixed by checking UltimMC availability first: if present, wait for the Minecraft connection (up to 120s); if not, log clearly that manual setup is needed and don't block agent startup on it.
2. **Wrong UltimMC executable** — `_create_portable_package()` in `packager.py` wasn't copying the UltimMC folder into the portable package at all, even though `_setup_ultimmc()` had already prepared it — so packaged agents fell back to some other UltimMC install instead of their own preconfigured one (correct Minecraft account, Forge instance, mods, data directories already set up). Fixed by explicitly copying the folder.

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
Using `-s <server> -l <instance> -o <agent_name>` handles account creation and mod registration automatically — this is the "preferred method," called out explicitly as such in the source doc.

## Audit findings (AUDIT_REPORT.md, dated May 21 2026)

22 issues found across Gradle/Java, Python, config, and docs. The one **critical** item:

- **Duplicate `Config` classes** — `py_backend/config.py` (canonical, imported by 8 files: main.py, packager.py, auto_connect_system.py, minecraft_launcher.py, auto_packager.py, chat_launcher.py, agent_spawner.py) and `py_backend/ai_core/config.py` (near-identical duplicate, imported only by `ai_core/__init__.py`). Risk: configuration drift between the two over time. Fix identified but not confirmed applied as of this ingest: delete the `ai_core` copy, repoint `ai_core/__init__.py` at the canonical one.

Also flagged: 3 duplicate `requirements.txt` entries (websockets was one), 2 Gradle inconsistencies between the client/server mod modules, 2 duplicate READMEs across the Java modules, 3 backward-compatibility wrapper/shim files, and 6 smaller structural issues. Not individually itemized here — see `AUDIT_REPORT.md` directly if working on any of these areas, since the full report has file:line references this page doesn't reproduce.

## A discrepancy this wiki itself surfaced

Not from any of the three source docs, but worth recording here since it c "roughly one-minute" BrainCapsule save cadence as a build target; the actual current code (`run_standalone_agent`'s main loop) saves every 300 seconds. See [[agent-runtime]] for the detail. Not a bug in either doc individually — the design wiki is describing a target for hardware that doesn't exist yet — but the numbers don't match today, and it's the kind of thing a lint pass across both wikis should keep catching.

## Deprecated, not a "problem" — just don't re-document as current

`py_backend/oracle_stuff/` (`create_oracle.py`, `teach_oracle.py`, `create_custom_texts.py`) — confirmed by Devlord as old and unused. See [[oracle-two-systems]] for the full context. Recorded here too since "old scripts still sitting in the repo" is exactly the kind of thing a future ingest pass might otherwise stumble on and assume is live.
