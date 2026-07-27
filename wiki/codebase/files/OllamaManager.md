---
type: file
status: ingested
---
# OllamaManager.java

💡 **Role**: the actual HTTP client to a local Ollama daemon (`http://127.0.0.1:11434`) — connection discovery, model listing/pulling, and text generation. Entirely static (no instance, module-level state via `static` fields). Part of [[oracle-two-systems]]'s System B, unrelated to the Python backend or `NPCAgent` in any way — this class's only network target is Ollama, never anything on port 11400.

## Imports

`com.divineworld.DWMod`, `com.google.gson` (`Gson`, `JsonArray`, `JsonObject`) — internal/external. `java.net` (`URI`, `http.HttpClient`, `http.HttpRequest`, `http.HttpResponse`), `java.time.Duration`, `java.util` (`ArrayList`, `List`) — stdlib.

## Fields

- `OLLAMA_HOSTS` — a one-element array, `127.0.0.1:11434` only; a commented-out IPv6/LAN address sits beside it with a note about removing a `user_jvm_args.txt` entry to re-enable it. Despite being an array (implying multi-host fallback was intended), only ever has one real entry in the current code.
- `OLLAMA_HOST` — resolved at `initialize()`/`refresh()` time, `null` until then.
- `HTTP_CLIENT` — one shared `HttpClient`, explicitly forced to HTTP/1.1 (`.version(HttpClient.Version.HTTP_1_1)`) rather than letting the JDK negotiate HTTP/2 — worth reading as a deliberate compatibility choice against Ollama's own server, not an oversight, given how specifically it's set.
- `GSON`, `initialized`.

## Methods

- `initialize(defaultModel)` — tries each host in `OLLAMA_HOSTS` (currently just one) with a `GET /api/tags`, sets `OLLAMA_HOST`/`initialized` on the first 200 response, logs the default model's availability status, and — on total failure — logs a specific, actionable troubleshooting checklist (`systemctl status ollama`, port check, manual `curl`, Java version, `/oracle restart`) rather than a bare error.
- `isOllamaRunning()` — a live ping (`GET /api/tags`) against the already-resolved host; returns `false` immediately if no host has been resolved yet, without attempting a request.
- `getAvailableModels()` / `isModelAvailable(name)` / `listAvailableModels()` (private, logging-only) — model listing and membership checks, all built on the same `GET /api/tags` call, none cached — every call is a fresh round-trip.
- `pullModel(name)` — `POST /api/pull`, a 30-minute timeout (explicitly commented as long, for model downloads).
- `generateWithOptions(model, prompt, temperature, contextTokens)` — the actual generation call, `POST /api/generate` with `stream: false` (a single JSON response rather than Ollama's line-delimited streaming format) and a 5-minute timeout. Re-runs all three pre-flight checks itself (redundant with [[LLMOracleBrain]]'s own pre-flight checks before calling this, but harmless — just a second, cheap confirmation) before sending. Distinguishes timeout, connection-refused, and malformed-JSON failures into three different thrown exceptions with different user-facing text, rather than one generic failure message.
- `getHost()` / `getStatus()` / `isInitialized()` — status accessors, `getStatus()` returning a Minecraft-formatted colored string (`§a`/`§c`) for direct display.
- `shutdown()` — just flips `initialized` false; nothing to actually close on an `HttpClient`.
- `refresh()` — re-runs the same host-discovery loop as `initialize()`, for the `/oracle restart` command.
- `checkModelStatus(model)` — returns an `enum ModelStatus` (`AVAILABLE`/`NOT_DOWNLOADED`/`OLLAMA_NOT_RUNNING`/`ERROR`); both "not initialized" and "ping failed" collapse to the same `OLLAMA_NOT_RUNNING` value rather than being distinguished — a minor conflation, not a bug, since both cases genuinely mean "nothing is available right now" to a caller.

## Problems (faced by traditional AI systems / LLMs)

Not AI/ML-specific — the interesting problem here is operational: talking to a local, potentially-not-running daemon whose failure modes (not started, still loading a model, wrong port, wrong HTTP version negotiated) all look similar from the outside but need different guidance to actually fix.

## Solutions

Distinguishing failure types by exception class (`ConnectException` vs `HttpTimeoutException` vs a bad-JSON parse) rather than catching one generic `Exception` everywhere is what makes the specific troubleshooting messages possible — the code knows _which_ thing went wrong, not just that something did. Forcing HTTP/1.1 explicitly, rather than letting version negotiation happen implicitly, removes one entire category of "works sometimes" failure before it can occur.

## Files Required

None beyond the JDK's own `java.net.http` client and Gson.

## Files Used In

- [[LLMOracleBrain]] — every generation call.
- [[OracleSystem]] — indirectly, via `LLMOracleBrain`; not called directly.