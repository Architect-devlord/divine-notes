---
type: file
status: ingested
---
# LLMOracleBrain.java

💡 **Role**: a thin async wrapper around [[OllamaManager]] — pre-flight checks (initialized/running/model-available), a shared busy flag so a teaching pass and a direct question never hit Ollama concurrently, and callback delivery back onto the Minecraft server thread. This is **System B** per [[oracle-two-systems]] — an entirely self-contained, Java-only, Ollama-backed system with zero connection to `NPCAgent`/`BrainCore`/the Python stack. **Resolves and corrects a thread from earlier this session**: `main.py`'s `/api/agents/chat_heard` docstring named this class as a possible explanation for [[chat_system]]'s `hasattr()` defensiveness around god agents' brains. Read directly, that doesn't hold — this class has nothing to do with god agents at all; it's the brain behind [[OracleSystem]]'s Wandering-Trader tutor NPC. Corrected on [[main]], [[chat_system]], and `index.md`.

## Imports

`com.divineworld.DWMod`, `net.minecraft.server.MinecraftServer` — internal/Minecraft. `java.util.concurrent` (`CompletableFuture`, `AtomicBoolean`), `java.util.function.Consumer` — stdlib.

## Fields

`modelName`, `endpoint` (both mutable, via `switchModel()`), `isSecondOracle` (final — the class supports at least two independent instances; the exact split between them wasn't traced in the source `oracle-two-systems.md` this pass inherited from, and wasn't chased down further here either), `temperature` (default 0.4), `contextTokens` (default 2048), `busy` (`AtomicBoolean`, the concurrency gate).

## Methods

- `queryAsync(server, prompt, callback)` — the primary entry point, called by [[OracleSystem]] for both direct `/oracle ask`-style questions and teaching-content generation (not the same as teaching-content _delivery_ — [[OracleSystem]]'s teach mechanic delivers pre-written material lines, not LLM-generated ones, per that file's own account; this method is for the interactive-question path). Three pre-flight checks (initialized, running, model available) each return early with a user-facing error message _before_ touching `busy` — a deliberate ordering documented in-line: a check failure never touched Ollama, so it shouldn't claim the resource. Runs the actual generation on `CompletableFuture.runAsync`, sets `busy` only after pre-flight checks pass, clears it in a `finally` block (so a thrown exception still releases it — otherwise one failed generation would permanently lock out every future caller, including the teaching loop), and delivers the result back via `server.execute()` so the callback runs on the correct (server) thread rather than the async worker thread.
- `query(prompt)` — a `CompletableFuture<String>`-returning alternative to the callback style above, same pre-flight checks and busy-flag discipline.
- `testConnection()` — a synchronous, non-generating check (initialized + running + model available), used for status commands.
- `isBusy()`, `switchModel()`, `setTemperature()`, `setContextTokens()` — plain accessors/mutators.

## Problems (faced by traditional AI systems / LLMs)

A shared, stateful external resource (one Ollama daemon) can only usefully serve one request at a time for a given model without either resource contention or misattributed output — and any caller that forgets to release a lock after a failure creates a permanent deadlock for every future caller, not just itself.

## Solutions

Centralizing the busy-flag management _inside_ this class's own `queryAsync()`/`query()`, rather than requiring every call site to remember to set and clear it themselves, is the real fix — confirmed by this class's own in-line comment as a deliberate design choice specifically because [[OracleSystem]] isn't the only caller (`OracleCommandRegistrar.java`, not yet read, calls `queryAsync()` directly too), so every current _and future_ caller sharing one `LLMOracleBrain` instance is covered automatically. The `finally`-block release is what makes this actually safe rather than just usually-safe.

## Files Required

- [[OllamaManager]] — every actual generation call.

## Files Used In

- [[OracleSystem]] — constructed once, held as a field, used for both direct questions and gating the teaching loop against concurrent generation.
- `OracleCommandRegistrar.java` (not yet read) — per this file's own in-line comment, calls `queryAsync()` directly for the `/oracle ask`-style command.