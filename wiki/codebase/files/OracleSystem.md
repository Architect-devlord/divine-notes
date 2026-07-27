---
type: file
status: ingested
---
# OracleSystem.java

💡 **Role**: the actual Wandering-Trader tutor NPC — spawn/despawn, per-player tutorial gating, and the `/oracle teach` mechanic. [[oracle-two-systems]] already covers this class's role and the teach mechanic in real depth, confirmed accurate against the source directly on this pass, including one specific claim worth calling out precisely because it's the kind of detail easy to get slightly wrong secondhand: teaching content really is delivered with **no special event type or flag of any kind** distinguishing it from ordinary overheard speech — confirmed in this file's own in-line comment, not just asserted.

## Imports

`com.divineworld.DWMod`, `com.divineworld.utils` (`BookFactory`, `DivineMagicCircle`), `com.divineworld.events.ProximityChatHandler`, `com.divineworld.integration.PythonBackendClient`, `com.divineworld.utils.AgentConfigLoader` — internal. `com.google.gson` (`Gson`, `TypeToken`) — external. A large Minecraft/Forge import block (entity, level, particle, sound, event types). `java.io.*`, `java.nio.file.*`, `java.util.*`, `java.util.concurrent.ConcurrentHashMap` — stdlib.

## Fields

Per-player state, all keyed by player `UUID`: `tutorialCompleted` (a `Set`), `activeOracles` (UUID → the spawned `Mob`), `lastInteractionTime`, `memoryMap` (UUID → `OracleMemory`, a private inner class holding a capped conversation history + last-access time). `brain` (one shared [[LLMOracleBrain]]), `personaTemplate` (a fixed system-prompt string: wise, mysterious, concise, 1-3 sentences). Reaction-circle state (a short visual burst on Genesis/Reset events, triggerable by external callers like `GenesisManager`/`DivineCommands`). Teaching state, six more UUID-keyed maps (`teachingRequested` as a `ConcurrentHashMap`-backed set — the one piece of state actually meant to be touched from off the server thread — the rest as plain `HashMap`s since they're only ever touched from the tick handler and command handlers, both on the server thread): current target agent, engagement start time, last-taught agent (excluded from the next round), material-line index, last-speech time. `TEACHING_ROTATION_MS` (20 real-world minutes), `TEACHING_SPEECH_INTERVAL_MS` (20 seconds — this file's own comment is explicit this cadence isn't specified anywhere upstream, just a reasonable default chosen for ~60 lines per engagement), `TEACHING_MATERIALS_REFRESH_MS` (5 minutes, for live-editing the materials folder without a restart).

## Constructor

`OracleSystem(LLMOracleBrain brain)` — creates the memory folder (`config/divineworld/oracle_memory/`) if missing, loads existing memory, and registers `this::handleChatHook` with `ProximityChatHandler.setChatHook()` — a settable hook this class uses to intercept a player's chat during their own active tutorial (checking for "I know"/"teach me") before, or instead of, that chat being processed as ordinary nearby-player speech. The exact semantics of what the hook's boolean return controls on the `ProximityChatHandler` side weren't traced this pass — flagged for when that file gets its own read.

## Key methods

- `spawnOracle(player)` / `despawnOracle(player)` — particle/sound spawn theater, an invulnerable, non-AI `WANDERING_TRADER` tagged `is_oracle` + `oracle_owner` (the owner's UUID, stored in persistent NBT data so the tick handler can find the right player later).
- `onServerTick()` — three things every tick: (a) continuously re-applies look-at rotation for every active oracle toward its owner (falling back to the nearest player within 32 blocks if the owner's disconnected) — the fix this file's own header describes as necessary because vanilla packet resets undo a one-shot look call; (b) advances the reaction-circle animation if one's active; (c) calls `processTeachingTick()`.
- `onPlayerJoin()` / `onOracleInteract()` — both gate on `AgentConfigLoader.getAgentTypeForName(name) != null`, excluding AI agents. The file's own header is explicit this is two separate fixes, not one: `onPlayerJoin()` already had the check; `onOracleInteract()` didn't, so an agent could walk up and right-click a _human's_ already-spawned oracle and still receive tutorial books, since that method only ever checked the target entity's tag, never who was doing the interacting.
- `handleChatHook(player, rawMsg)` — checks for `"i know"` (skip tutorial, schedule a confirmation message) among the tutorial-flow phrases; returns a boolean signaling whether this message was consumed.
- **The teach mechanic**, four cooperating private methods called from `processTeachingTick()`: `discoverTeachableAgent()` (scans live players, keeps only ones `AgentConfigLoader` classifies as agents, excludes whoever was just taught), `engageAgent()` (teleports the oracle to within a fixed 2-block offset of the target, snapped to a safe standing position — deterministic, not pathfinding-based, explicitly noted as sufficient for the "within 3 blocks" requirement without needing more), `deliverTeachingTick()` (keeps facing the target, delivers one cached teaching-material line every 20s via **`PythonBackendClient.notifyChatHeard(agentName, "Oracle", line)` — the exact same call `ProximityChatHandler` uses for real nearby speech**, confirmed directly, no special-casing anywhere in the payload), and the 20-real-world-minute rotation check inline in `processTeachingTick()` itself (wall-clock `System.currentTimeMillis()`, not game ticks).
- Teaching-materials folder resolution — its own in-line comment states directly that it mirrors [[mc_uuid]]'s `AgentNameManager._find_config_path()` fallback chain exactly (Documents → Desktop → OneDrive-redirected equivalents of each → auto-create Documents as a last resort) — a genuine, deliberate, named instance of cross-language design parity between this Java file and that already-documented Python one, not a coincidence.
- `runTutorial()`, `saveMemory()`/`loadAllMemory()`, `getSafeSpawnPosition()`, `setOracleBrain()`/`getOracleBrain()` — per this file's own header comment, unchanged from an earlier version; not independently re-verified beyond confirming they exist and are called where expected.

## Problems (faced by traditional AI systems / LLMs)

Not AI/ML-specific from this class's own side — the interesting problem is a game-engine one (an NPC needing to persistently face a moving target despite the engine resetting rotation every tick) and a design one (injecting curated content into an autonomous agent's experience without the agent being able to distinguish it from organic input, which is a deliberate design goal here, not an accident).

## Solutions

Re-applying look-at every tick rather than once at spawn is a direct, correct fix for the engine-reset problem. The "no distinguishing flag" design for delivered teaching content is a genuine, considered choice, not an oversight — confirmed by the in-line comment's own framing ("the agent has no way to know it is being taught rather than just being spoken to") — whether that's the _right_ choice for training data quality is a separate question this ingest doesn't take a position on, but it's clearly intentional.

## Files Required

- [[LLMOracleBrain]] — held as a field, used for both direct questions and the teaching-loop concurrency gate.
- `AgentConfigLoader.java` (not yet read) — `getAgentTypeForName()`, called at three separate points (`onPlayerJoin`, `onOracleInteract`, `discoverTeachableAgent`) to distinguish real players from AI agents.
- `PythonBackendClient.java` (not yet read) — `notifyChatHeard()`, the teach mechanic's actual delivery call into the Python backend. **This call target is fixed as of the 2026-07-16 pull** (was confirmed broken when this page was first written; confirmed fixed now by diff, not assumed): `/api/agents/chat_heard` no longer calls a nonexistent method — it now forwards to the target agent's own `/api/perception/chat_heard` route directly, so `/oracle teach` delivery should genuinely work again. See [[main]] for the full fix.
- `BookFactory.java`, `DivineMagicCircle.java`, `ProximityChatHandler.java` — none read yet this pass.

## Files Used In

- Constructed once, presumably in `DWMod`'s server-start sequence alongside [[LLMOracleBrain]] — not yet confirmed directly, since `DWMod.java` hasn't been read this pass.
- `GenesisManager.java`/`DivineCommands.java` (not yet read) — per this file's own header comment, external callers of the reaction-circle trigger.
- `OracleCommandRegistrar.java` (not yet read) — presumably the actual `/oracle ...` command implementations calling into this class's public methods.