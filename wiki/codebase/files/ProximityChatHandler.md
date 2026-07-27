---
type: file
status: ingested
---
# ProximityChatHandler.java

💡 **Role**: replaces Minecraft's global chat broadcast with proximity-scoped delivery, and is the confirmed real source of the NPC/god chat-routing split already inferred from [[main]]'s docstring — read directly here, with one real correction to how that split actually works.

## Imports

`com.divineworld.DWMod`, `com.divineworld.integration.PythonBackendClient`, `com.divineworld.utils.TaggedEntitySystem` — internal. `net.minecraft.network.chat.Component`, `net.minecraft.server.level.ServerPlayer`, `net.minecraft.world.entity.player.Player` — Minecraft. `net.minecraftforge.event.ServerChatEvent`, `eventbus.api.SubscribeEvent`, `fml.common.Mod` — Forge.

## Fields

`PROXIMITY_RADIUS = 10.0` (blocks, 3D Euclidean). `chatHook` — a static, settable `ChatHook` functional interface (`boolean onChat(ServerPlayer sender, String rawMessage)`), registered by [[OracleSystem]] in its own constructor.

## The Oracle-interception fix

Documented directly in the class docstring: this handler used to call `event.setCanceled(true)` immediately, which meant [[OracleSystem]]`.onPlayerChat()` — a normal `@SubscribeEvent` listener — never saw the event at all, since Forge skips cancelled-event listeners by default (`receiveCancelled=true` is opt-in, not automatic). The fix: rather than relying on Forge's own listener-ordering semantics, this handler calls the registered hook **directly, as a plain method call**, always first, before making any cancellation decision — sidestepping the ordering problem entirely instead of trying to solve it within Forge's event system.

## `onServerChat(event)` — the actual routing logic

1. The Oracle hook runs first, unconditionally, wrapped in its own try/catch so a hook exception can't break chat entirely.
    
2. **The override-scope decision, worth being precise about since it's more selective than "always intercept everything"**: if the sender isn't an AI agent _and_ no AI agent is within radius, this handler does nothing beyond the hook — vanilla global broadcast proceeds completely unmodified. Real-player-to-real-player chat with no agents anywhere near is untouched by this whole system.
    
3. Otherwise, the vanilla broadcast is cancelled and this handler does its own delivery: builds a `<Sender> message` component, sends it to the sender and every same-dimension player within radius, and — **the confirmed, precise version of the NPC/god split**:
    
    > _"Notify ALL AI agent recipients (both NPC and GOD) via HTTP so the Python cognitive loop can react immediately. NPC agents also receive it via `ClientChatReceivedEvent` on their own Minecraft client — the HTTP path gives an immediate channel without waiting for the next perception frame. GOD agents rely solely on HTTP (no standard chat listener on their client)."_
    
    Every in-range NPC or god recipient gets `PythonBackendClient.notifyChatHeard()` — **not just gods**. The distinction isn't "NPCs use one channel, gods use another" (which is how [[main]]'s own docstring, and this wiki's earlier read of it, summarized the split) — it's "gods use HTTP only; NPCs use HTTP _and_ an additional client-side event channel." **Correction to [[WebSocketManager]]**: that page speculated `sendChatObservation()` was "almost certainly" what this handler calls for NPCs specifically. Checked directly against this file — it isn't. This handler never touches `sendChatObservation()` at all; it uses `PythonBackendClient.notifyChatHeard()` (HTTP) uniformly for every agent type. Where `sendChatObservation()` actually gets called from, if anywhere, is unconfirmed — flagged as a genuinely open thread rather than left as the wrong guess.
    

## Helpers

- `hasAgentNearby(player)` — same-dimension, same-radius scan used only to decide whether a real player's own message needs proximity scoping at all.
- `buildMessage()` — plain `<Name> message` formatting, matching vanilla's own display convention.
- `distanceSq()` — squared distance, compared against `PROXIMITY_RADIUS²` rather than taking a square root on every chat message; the in-line comment tags this `FIX HF-3` explicitly as a performance fix, not incidental.
- `sameDimension()` — plain dimension-key equality check.

## Problems (faced by traditional AI systems / LLMs)

Not AI/ML-specific — the interesting problem here is a Forge event-system one, general to any plugin/mod architecture with cancellable events and multiple independent listeners: cancelling an event early can silently prevent other legitimate listeners from ever running, in a way that's easy to miss until a specific feature (Oracle interception) depends on both running.

## Solutions

Calling the registered hook as a direct method invocation, rather than depending on Forge's cancelled-event listener semantics, removes the ordering dependency entirely — it doesn't matter what order Forge would have called listeners in, because this class controls the order itself. The proximity-radius squared-distance optimization is a small, correct, low-risk win: avoiding a `sqrt()` call on a code path that runs on every single chat message in a populated world adds up in a way a one-off calculation wouldn't.

## Files Required

- `com.divineworld.integration.PythonBackendClient` — `notifyChatHeard()`. See [[PythonBackendClient]].
- `com.divineworld.utils.TaggedEntitySystem` — `detectAgentType()`, `AgentType` enum (not yet read).
- `com.divineworld.entity.DWNPCManager` — `isAIPlayer()` (not yet read).

## Files Used In

- [[OracleSystem]] — registers `chatHook` in its own constructor, confirmed matching this file's `setChatHook()` signature exactly.
- [[PythonBackendClient]]'s own javadoc on `notifyChatHeard()` names this file as its caller — confirmed accurate, both directions agree.