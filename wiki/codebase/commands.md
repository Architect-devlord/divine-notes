---
type: system
status: ingested
---

# Server Commands

💡 **What this is**: every `/command` DivineWorld registers, and the one dispatcher they all now go through. `BreedCommand` and `CraftCommand` already have their own coverage in [[breeding-system]]/[[crafting-system]]; this page is everything else.

## One entry point, a fixed bug behind it

`CommandRegistrar.register()` is the only thing `DWMod`'s command-registration hook calls now, in turn calling all five command groups (`DivineCommands`, `NPCCommand`, `GodCommand`, `OracleCommandRegistrar`, `BreedCommand`, `CraftCommand`) in order. Worth knowing this consolidation is itself a fix: `CommandRegistrar` used to exist but was never actually called from anywhere — `DWMod` called each group directly, one by one, leaving this class fully dead despite being imported. Adding a new command group now only needs one new line here.

## `DivineCommands` — the core admin/genesis surface

`/genesis` (cooldown-gated via [[known-issues|GenesisManager]], spawns the first two agents), `/divine_reset`, `/clear_memories <all|agent_id> [exceptions...]`, `/spawn_god <type>`, `/god_ability <agent_id> <ability>`, `/god_transform <mob>` / `<agent_id> <mob>` / `<agent_id> revert`, `/list_agents`.

The one worth remembering: `/god_ability`'s tab-completion list is convenience only. `executeGodAbility()` passes whatever string was typed straight through to `ServerGodAbilityExecutor.execute()` with no whitelist of its own — an operator who already knew an ability name (e.g. `summon_vexes` before it was added to the suggestion array) could always call it; only autocomplete was ever missing it.

## `GodCommand` — `/godtoggle`, three ways

One command, three arities, each meaning something different:
- **No args** — cycle the *caller's own* form (`GodDisguiseHandler.cycleGodForm`), gods only.
- **One arg** — tries the argument as an online god's display name first (cycles *that* god's form instead); if no god matches, falls back to treating it as a mob type for a self-transform. That fallback is flagged in the class's own comment as a rare, mostly-scripted path kept only for backward compatibility — `/god_transform` is the normal way to do a mob transform.
- **Two args** (`<agent_id> <mob>`) — admin transforms a *specific* named god into `<mob>`, or reverts it. Unchanged behavior from before the single-arg god-cycling feature was added.

Also documents, in its own header comment, exactly how the Creaking's asset split stays transparent to the renderer: in "god" (true) form it renders via `ai_creaking.*` through `CreakingGeoRenderer`; in "humanoid" form it uses `god_creaking.*` like every other god, because `GodHumanoidGeoModel.getModelResource()` always asks for `"god_" + godType` regardless of which god it's rendering — the split is handled once, generically, not per-god.

## `NPCCommand` — `/dw npc spawn|list|remove|info`

Plain CRUD for non-god NPCs specifically (`list`/`remove`/`info` all explicitly skip god agents — `/divine_reset` is the intended way to remove a god). `spawn` calls `PythonBackendClient.spawnSingleAgent()` — its own comment flags this as a fix, replacing an earlier version that reused the multi-agent genesis endpoint for a single spawn. `getSafeSpawnPosition()` walks upward from the target block up to 10 times looking for an air/air/solid three-block window before giving up and using whatever position it landed on.

## `OracleCommandRegistrar` — the Oracle's full lifecycle, in one tree

Eleven subcommands under `/oracle`: `spawn`, `despawn`, `teach` / `stop_teach` (sets intent only — the class comment ties this to `OracleSystem`'s own next-tick gating on tutorial-completion + Ollama-availability, not registered here), `set_model` / `list_models` / `test`, `status`, `refresh`, `pull <model>` (runs on a background thread, reports back via `server.execute()` so the chat message lands on the main thread), `restart`, and `help`. `AVAILABLE_MODELS` (`phi3:mini`, `gemma3:1b`, `deepseek-r1:8b`, `llama2`, `mistral`) is purely a tab-completion list — `set_model` accepts any string, so an unlisted model name still works if Ollama has it.
