---
type: file
status: ingested
---
# PythonBackendClient.java

💡 **Role**: the server mod's outbound HTTP client to [[main]]'s FastAPI app — fire-and-forget, one method per event type, every one of them building a JSON payload and handing it to a shared `sendAsync()` helper. **A genuinely comprehensive confirmation file**: every method here maps onto a route already documented on [[main]] this session, closing the loop on "is this route actually called from the Java side" for all of them at once, rather than leaving it inferred.

## Imports

`com.divineworld.DWMod`, `com.google.gson` (`JsonArray`, `JsonObject`), `net.minecraft.core.BlockPos` — internal/Minecraft. `java.net.http.*`, `java.util.List` — stdlib.

## Fields

`BACKEND_URL` — `http://127.0.0.1:11400` by default, overridable via `-Ddw.backend`. `HTTP_CLIENT` — one shared `HttpClient.newHttpClient()`.

## Methods

Every method builds a `JsonObject`, adds a `timestamp`, and calls `sendAsync(json, endpoint)`. Confirmed destinations, all already documented on [[main]]:

- `spawnSingleAgent(agentName, spawnerName, worldName, spawnPos)` → `/api/agents/spawn_single`.
- `spawnGenesisAgents(spawnerName, worldName, spawn1, spawn2)` → `/api/genesis/spawn`, hardcoding the two spawn positions' genders `male`/`female` respectively — matching [[main]]'s own documented "Adam"/"Eve" ancestor convention.
- `notifyDivineReset(worldName, agentIds)` → `/api/divine_reset`.
- `clearAgentMemories(agentIds, exceptions)` → `/api/agents/clear_memories`, pre-filtering `exceptions` out of the target list client-side before sending, rather than relying on the Python side to do the exclusion.
- `spawnGodAgent(godType, spawnerName, worldName, spawnPos)` → `/api/gods/spawn`.
- `godUseAbility(agentId, abilityName, params...)` → `/api/gods/ability`. **Confirms a real, live caller exists for a route already confirmed as an unimplemented stub** — [[main]] documented this endpoint as echoing its input back and touching nothing real; this method is presumably wired to a `/dw god ability`-style human operator command, not the AI's own action-frame ability triggering (a separate, already-resolved mechanism — see [[ServerGodAbilityExecutor]]). Worth being precise about the distinction: this is a human-operator HTTP path that currently does nothing; the AI's own ability triggering is live and goes through an entirely different channel.
- `godTransform(agentId, targetMob)` → `/api/gods/transform`. Same situation — a real caller for the other confirmed stub.
- `notifyBreeding(parentAId, parentBId, parentAType, parentBType)` → `/api/breeding/event`.
- `notifyChatHeard(hearerAgentId, speakerName, message)` → `/api/agents/chat_heard`. Its own javadoc names the caller directly: _"Called by ProximityChatHandler for every agent within PROXIMITY_RADIUS."_ Confirms [[main]]'s already-fixed forwarding mechanism and [[OracleSystem]]'s `/oracle teach` dependency both reach exactly this method.
- `sendAsync(json, endpoint)` — the shared send path. **Spawns a brand new `Thread` per call**, not a shared executor or thread pool — every single backend notification gets its own throwaway thread. A generous 600-second timeout, with an in-line comment explicitly justifying it ("agent spawning, breeding, executable generation" — all genuinely slow operations documented elsewhere this session, e.g. [[packager]]'s own multi-hundred-second polling ceilings). Only the HTTP status code is ever read from the response — the body is discarded entirely, so this class has no way to surface _why_ a call failed, only that it did (or a 200, that it didn't).

## Problems (faced by traditional AI systems / LLMs)

Not AI/ML-specific — a plain fire-and-forget integration problem: this class can tell a caller "sent" or "logged an error," never "here's what actually happened on the other end." For the two stub routes above, that means a human operator calling `/dw god ability` sees a success log from this class even though nothing real happened on the Python side — the failure is invisible at exactly the layer someone would be watching.

## Solutions

Not solved in this file — genuinely fire-and-forget by design, appropriate for truly one-way notifications (breeding events, memory clears) but a real blind spot for anything where the caller might want to know the request didn't do what it looked like it did. The per-call `Thread` spawn, while not the most efficient possible pattern, does mean one slow backend call (up to 600s) never blocks another — a real, if blunt, isolation guarantee.

## Files Required

None beyond Gson and the JDK's own `java.net.http` client.

## Files Used In

- Presumably various command classes (`DivineCommands.java`, `GodCommand.java`, `NPCCommand.java`, `BreedCommand.java` — none read yet this pass) for the human-operator-triggered methods.
- [[OracleSystem]] — `notifyChatHeard()`, confirmed directly (that file's own `/oracle teach` delivery call).
- [[ProximityChatHandler]] (next in this batch) — `notifyChatHeard()`, per this file's own javadoc naming it as the caller.