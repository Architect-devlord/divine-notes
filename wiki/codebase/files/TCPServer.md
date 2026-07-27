---
type: file
status: ingested
---
# TCPServer.java

💡 **Role**: the TCP fallback action channel — a `ServerSocket` this client listens on, accepting one Python connection at a time and parsing the same conceptual action frame [[WebSocketManager]] handles, but over plain TCP rather than a WebSocket. Its own class docstring names the Python-side counterpart directly: `ForgeIPCClient.send_action()` in `actuators.py` — **a separate Python sender from `communication_protocol.py`'s WebSocket-based `BinaryProtocol`**, not yet cross-checked against that file's own already-documented behavior, but a concrete, named lead for when `actuators.py` gets its own re-audit (already flagged as owed, from the 2026-07-16 reconciliation).

**Confirms, and then escalates, [[WebSocketManager]]'s flagged finding**: the class docstring states plainly, and the code matches exactly, that any ability string not prefixed `inv:`/`screen:` is "god ability dispatched to `GodEntityManager.executeGodAbility()`" — the same method [[WebSocketManager]] calls. **Confirmed dead as of 2026-07-16**, not just client-visual-only as first thought: a full read of [[GodEntityManager]] found its dispatch depends on a `currentGodEntity` field that's never assigned anywhere in the repository. Confirmed now across _both_ ingest paths — this file and [[WebSocketManager]] both hand off to the same confirmed-inert method. Whatever connects an AI agent's own ability decision to the already-confirmed-correct server-side [[ServerGodAbilityExecutor]] isn't this file's concern, and isn't currently working through either networking path.

## Imports

`com.divineworld.client.DWClientMod`, `com.divineworld.client.control.ActionExecutor` — internal. `net.minecraft.client.Minecraft`, `net.minecraft.world.inventory.ClickType` (imported, unused in the visible logic — a type reference for documentation purposes in the class docstring's inventory-action examples, not actually referenced in code) — Minecraft. `java.io.DataInputStream`, `java.net.ServerSocket`/`Socket`, `java.nio.charset.StandardCharsets`, `java.util.concurrent.*` — stdlib.

## Wire format (from the class docstring, cross-checked against the parsing code directly)

Agent ID (length-prefixed UTF-8) → tick (int64, read but discarded) → four movement floats → one packed flags byte → hotbar slot (`0xFF` = no change) → a length-prefixed ability section (name + three trailing floats, only present if length > 0) → a length-prefixed chat section (same convention, only present if length > 0). The chat section is explicitly flagged in-line as a fix: **"chat mapping gap - always present, same 2-byte-length-prefix convention as the ability section... read unconditionally."** This is the Java-side counterpart to the exact `communication_protocol.py` chat-mapping fix already noted (but not yet fully reconciled) from the 2026-07-16 pull's diff — this file already reflects the fixed state, since it's being read fresh rather than needing its own correction.

## Methods

- `start(port)` / `runServer(port)` — a single dedicated daemon thread runs an accept-loop wrapping an inner read-loop; one client at a time (`clientSocket` is a single field, not a collection) — this is a fallback channel for one agent's own dedicated connection, not a multi-agent server.
- `handleActionFrame()` — the parser described above. Dispatches everything onto the Minecraft main thread via `Minecraft.getInstance().execute()`, in a fixed order: (1) movement/flags via [[ActionExecutor]]`.executeAction()`, unconditionally; (2) chat via `executeChatAction()`, if present — explicitly independent of the ability slot, so an agent can speak and act in the same frame; (3) the ability string, routed three ways: `inv:` → `executeInventoryAction()`, `screen:` → `executeScreenAction()`, anything else → `GodEntityManager.executeGodAbility()`, wrapped in a try/catch that only logs at debug level on failure — a god-ability dispatch error here is deliberately non-fatal to the rest of the frame.
- `stop()`/`cleanup()` — closes stream/socket/server-socket in that order, each independently guarded so a failure closing one doesn't skip the others.
- `isConnected()` / `getPort()` — plain status reads.

## Problems (faced by traditional AI systems / LLMs)

Not AI/ML-specific — maintaining two independent wire implementations (this file, [[WebSocketManager]]) for the same conceptual action frame is a real, general risk: any change to the frame's shape has to be made twice, correctly, in two unrelated classes, or the two channels silently diverge. The chat-section fix landing in this file (matching a fix already seen in `communication_protocol.py`'s own diff) is a concrete instance of exactly that kind of divergence having existed and needed correcting.

## Solutions

Not solved architecturally in this file — there's no shared frame-parsing code between this class and [[WebSocketManager]], so the risk above remains structural, not resolved. What is a genuine, working solution here: routing every dispatch through `Minecraft.getInstance().execute()` regardless of which thread the socket read happened on, exactly mirroring [[WebSocketManager]]'s own main-thread discipline — the two channels may duplicate parsing logic, but they agree on the more safety-critical rule of never touching game state off the main thread.

## Files Required

- [[ActionExecutor]] — `executeAction()`, `executeInventoryAction()`, `executeScreenAction()`, `executeChatAction()`.
- `GodEntityManager.java` (client-side, not yet given its own page) — `executeGodAbility()`.

## Files Used In

- Started presumably from `DWClientMod`'s own startup sequence, likely as an alternative to [[WebSocketManager]] rather than alongside it — not yet confirmed which mode selects which channel.
- `actuators.py`'s `ForgeIPCClient.send_action()` (Python side, not yet read this pass) — the confirmed, named sender this file's own docstring points to.