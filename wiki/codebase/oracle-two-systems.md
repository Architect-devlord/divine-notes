---
type: system
status: ingested
---
# The Two Oracles

💡 **Why this page exists**: "Oracle" refers to two genuinely unrelated systems in this codebase, and they're easy to conflate — different files, different languages, different purposes, sharing only a name and a loose thematic role. Confirmed directly by Devlord. This page exists specifically so that doesn't happen again in future ingests.

## System A — the Oracle God Agent

A completely ordinary `NPCAgent` (see [[agent-runtime]]) with `god_type="oracle"`. Same `BrainCore`/`WorldModel`/`Personality`/`EmotionSystem`/policy stack as every other god — `Warden`, `Wither`, etc. — with `GodTransformerPolicy`'s 18-dim action space and whatever abilities `GodControlSystem("oracle")` defines. Spawned the same way any god is: through the standard `/api/gods/spawn` path.

**This is the Oracle documented in the design wiki** — [[wiki/design/Robots|Robots]] (via the Terminology section) and the dedicated [[wiki/design/Robots/Oracle|Oracle]]/[[Oracle_Archive]] pages there are entirely about this system: the physical robot body concept, the collective-perception idea, the staff-as-antenna design. Nothing on this page changes any of that.

### `py_backend/oracle_stuff/` — deprecated, not the current path

`create_oracle.py`, `teach_oracle.py`, and `create_custom_texts.py` implement an early experiment: spawning a `god_type="oracle"` agent and feeding it a bilingual modern↔Shakespearean/medieval-English corpus through the standard `brain_language` system, aiming to give it an archaic speech style.

**Confirmed by Devlord as old, outdated, and not part of the current system.** Recorded here for historical accuracy (the files still exist in the repo) but explicitly *not* as how the Oracle God Agent currently gets built or trained — treat it as dead code, not documentation of current behavior. If a current equivalent exists, it hasn't been identified yet; don't assume one and don't re-ingest `oracle_stuff/` on a future pass without a specific reason to revisit it.

## System B — the personal Ollama tutor ("the ollama one just for me")

An entirely separate, Java-side system: `OracleSystem.java`, `LLMOracleBrain.java`, `OllamaManager.java` (all in `DivineWorld/src/main/java/com/divineworld/oracle/`). Nothing here touches `NPCAgent`, `BrainCore`, or any of the Python AI stack at all.

**What it actually is**: a Minecraft "Wandering Trader"-disguised NPC that spawns automatically per-player (excluding AI agents themselves — `AgentConfigLoader.getAgentTypeForName()` is checked specifically to prevent an agent from wandering up and triggering another player's oracle). It runs a one-time tutorial for that player, and can answer direct questions by querying a local Ollama instance (`http://127.0.0.1:11434`) via plain HTTP — `OllamaManager.generateWithOptions(model, prompt, temperature, contextTokens)`.

**Key mechanics:**
- `LLMOracleBrain` takes an `isSecondOracle` flag at construction — the code supports (at least) two independent brain instances, e.g. potentially different roles or models, though the exact split wasn't fully traced in this pass.
- A shared `busy` `AtomicBoolean`, checked automatically inside `queryAsync()`/`query()` rather than by each caller, gates the system so a teaching pass and a direct question never compete for the same Ollama backend at once — deliberately centralized so every current *and future* caller is covered without having to remember to wrap itself.
- **`/oracle teach`** sends a player's already-spawned oracle wandering among live AI agents, delivering content from `~/Documents/DivineWorld/teaching_materials/` — but not as a special system message. It goes through `PythonBackendClient.notifyChatHeard()`, landing in the target agent's perception as an ordinary `chat_heard` event, **indistinguishable from a real player speaking nearby, by design**. Rotates to a different agent every 20 real-world minutes (wall-clock based, not tick-based). `/oracle stop_teach` clears it.

This is the "just for me" system — a personal dev/player-facing tool built on an off-the-shelf LLM playing a fixed wise-Oracle character, unrelated to the God Agent framework and not something the design wiki's Oracle pages are describing.

## The one place they could interact

`/oracle teach`'s perception-injection mechanic means System B *can* feed content to a System A agent (or any agent) — but only as an ordinary chat event indistinguishable from a human talking. System A doesn't know or care that the "speaker" was Ollama-generated. They remain architecturally separate; this is a one-directional injection point, not a merge of the two systems.

## Practical rule for future ingests

If a source or question says "the Oracle" without qualification, check which one is meant before writing anything down. Physical robot / God Agent / collective perception / staff antenna → System A, and the design wiki's [[wiki/design/Robots/Oracle|Oracle]] is the primary reference. Wandering Trader / tutorial / Ollama / `/oracle` commands → System B, and this page is the primary reference.
