---
type: file
status: ingested
---
# AgentConfigLoader.java

💡 **Role**: reads `agents.json` and classifies a display name as `NPC_MALE`/`NPC_FEMALE`/`GOD`/real-player — the shared identity source [[OracleSystem]] and [[GodDisguiseHandler]] both depend on. **Closes the loop on all four confirmed call sites found across this whole batch**: [[OracleSystem]]'s three (`onPlayerJoin`, `onOracleInteract`, `discoverTeachableAgent`) and [[GodDisguiseHandler]]'s one (`removeTransform`'s fallback chain) — all real, all confirmed matching this file's actual method signatures.

## Imports

`com.divineworld.DWMod`, `com.google.gson.*` — internal/external. `java.io.FileReader`, `java.nio.file.*`, `java.util.*` — stdlib.

## Fields

`CONFIG_FILE_NAME = "agents.json"`, `GSON`, `CACHE_DURATION_MS = 30_000` — a 30-second cache, avoiding a file re-read on every single classification check while still picking up changes reasonably quickly. `cachedConfig`/`lastLoadTime` — the cache state itself.

## File discovery

`findConfigFile()` — its own comment states it "mirrors `AgentNameManager._find_config_path` in `mc_uuid.py`," and does so with explicit OS branching not confirmed either way for the Python original this pass: Windows gets four candidates (`Documents`, `Desktop`, `OneDrive/Documents`, `OneDrive/Desktop`); macOS/Linux get two (`Documents`, `Desktop` only, no OneDrive-equivalent candidates, which makes sense given OneDrive path redirection is Windows-specific). First existing regular file wins; returns `null` (not an exception) if none exist, and the loader falls back to an empty config with a clear warning rather than failing — agents just get tagged by username prefix (`DW_`/`DWGOD_`) alone until Python's own `main.py` creates the file on its first startup.

## Parsing — a fixed historical bug, not a live one

The header docstring documents `S-01` directly: the old parser called something checking `isJsonArray()` on what `mc_uuid.py` actually writes as a `{"Name": port, ...}` JSON _object_ — always false for that shape, so every name list came back empty and every connecting agent was misclassified as a real player. **Already fixed in current code**, confirmed by reading `extractNames()` directly: it branches on `isJsonObject()` (keys are names, values — the ports — are read but discarded) versus `isJsonArray()` (a legacy flat-array format, still accepted with a logged warning rather than rejected outright). `parseConfig()` applies this uniformly across `NPCs.male`/`NPCs.female`/`GODs.dual.<type>`.

## Public API

- `loadConfig()` — the cache-checked entry point every other method routes through.
- `reloadConfig()` — forces cache invalidation.
- `isMaleNPC()`/`isFemaleNPC()`/`isGodName()`/`isNpcName()` — plain membership checks.
- `isValidGodType(key)` — per its own doc comment, used by `DivineCommands.executeSpawnGod` (not yet read) to validate a `/spawn_god <type>` command argument.
- `getGodTypes()` — every god-type key present in the loaded config.
- `getGodTypeForName(name)` — reverse lookup, name → type key; the exact method [[GodDisguiseHandler]]`.removeTransform()` calls as the second link in its fallback chain.
- `getAgentTypeForName(displayName)` — the classification method [[OracleSystem]] calls three separate times; returns `null` (not a default) when the name matches nothing, meaning "real player, or a freshly-packaged agent not yet registered" — genuinely ambiguous between those two cases from this method's own information alone.
- `AgentConfig` (public static inner class) — the parsed, immutable-view result: two name lists, a god-type list, and a `Map<String, List<String>>` of god names by type; the actual data `loadConfig()` returns and every convenience method above reads from underneath the static-method facade.

## Problems (faced by traditional AI systems / LLMs)

Not AI/ML-specific — the general problem is the same one this whole session has found repeatedly on both sides of the Python/Java boundary: two independently-maintained parsers for the same file format can silently disagree about that format's actual shape, and the disagreement only surfaces as a behavioral symptom (every agent misclassified as a real player) rather than a parse error, since the JSON itself was perfectly valid — just not the shape one side assumed.

## Solutions

`extractNames()` handling both the canonical object-keyed format and a legacy array format, rather than assuming only one, is a real, working fix — and a conservative one, since it doesn't require every historical `agents.json` file to be migrated before this code can read it correctly. The 30-second cache is a plain, appropriate tradeoff: frequent enough that a Python-side registration shows up quickly, infrequent enough that this file isn't re-read on every single chat message or entity check that needs a classification.

## Files Required

None beyond Gson and the JDK's own file/path APIs.

## Files Used In

- [[OracleSystem]] — `getAgentTypeForName()`, three confirmed call sites.
- [[GodDisguiseHandler]] — `getGodTypeForName()`, one confirmed call site.
- `DivineCommands.java` (not yet read) — `isValidGodType()`, per this file's own doc comment.

---

**This closes the Java-side "Python-dependent files first" priority batch.** Every file identified by the original search — Oracle package, god-ability routing chain, core wire-protocol files, HTTP-and-shared-config files — now has its own page. Remaining: the full Java-side sweep (~55 files across both mods not yet touched), per the original two-phase plan.