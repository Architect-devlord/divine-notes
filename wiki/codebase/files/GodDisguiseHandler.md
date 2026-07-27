---
type: file
status: ingested
---
# GodDisguiseHandler.java

💡 **Role**: the actual mob-disguise mechanic — swap a god-controlling player's visible model to a different mob entity, and back. Confirmed, per [[ServerGodAbilityExecutor]]'s own dispatch, as the real destination for `transform_<mob>`/`revert` — the file [[god-abilities]]'s open question was ultimately asking about. Contains a second, genuinely separate mechanism worth not conflating with the first: a three-way "form cycle" (boss body ↔ humanoid puppet ↔ disguise), controlled by its own `dw_form` NBT key, unrelated to which mob (if any) the player is currently disguised as.

## Imports

Minecraft/Forge entity, level, and NBT types. `com.divineworld.DWMod`, `com.divineworld.network.NetworkHandler`, `com.divineworld.utils.AgentConfigLoader` — internal.

## Methods: mob transform/revert

- `applyTransform(player, mobType, level)` _(public static)_ — works for both a god agent and a real, op-level-4 human player (both paths visible in the same method body, not two separate entry points — strong evidence this really is meant to be triggered from one shared place, matching [[ServerGodAbilityExecutor]]'s single-dispatch-point design). Spawns the target mob type, hides the player's own rendering, stores enough state (`dw_disguised`, the mob type, presumably a reference back to the original entity) to reverse it later, sends the player a confirmation message that explicitly names `/god_transform revert` as how to undo it manually. Calls `NetworkHandler.broadcastMorph()` at the end so nearby players' clients see the new appearance.
- `removeTransform(player)` — the inverse. Resolves which god type to restore to via a fallback chain: a stored `dw_original_god_type` NBT value first, then `AgentConfigLoader.getGodTypeForName(player's name)` (the third confirmed call site for that lookup this session, after two in [[OracleSystem]]), then a hardcoded `"oracle"` as a last resort if neither resolves. Also calls `NetworkHandler.broadcastMorph()` to sync the reversion.

## Methods: the form cycle (separate mechanism)

- `cycleGodForm(player)` / `applyGodForm(player, form)` / `getGodForm(player)` — a three-state cycle (`GOD` → `HUMANOID` → `DISGUISE` → back to `GOD`) stored in its own `dw_form` NBT key, independent of the mob-transform state above. This toggles _how_ the god-controlling player is rendered (its own boss-model body entity, a GeckoLib humanoid puppet, or a plain Steve/Alex-skinned disguise) rather than _what mob_ it's disguised as. Not the mechanism [[god-abilities]]'s open question was about — that question was specifically about `transform`/`revert` as _ability names_ reachable through the action-frame channel, and this form cycle wasn't found to have any connection to that dispatch path this pass. Worth a look from [[forms-and-disguise]] if that page doesn't already cover it, since this file is where it actually lives.

## Problems (faced by traditional AI systems / LLMs)

Not AI/ML-specific — a fairly ordinary game-state-representation problem: giving one entity multiple, independent axes of "what it looks like right now" (which mob, if disguised at all; which of three rendering modes) without the axes interfering with each other, and making both reversible from stored state rather than one-way.

## Solutions

Keeping the two axes (mob-disguise state, form-cycle state) in separate NBT keys rather than one combined enum is what keeps them genuinely independent — a player can be mid-transform-into-a-warden and still cycle their form representation without the two systems needing to know about each other. The fallback chain in `removeTransform()` (stored original type → config lookup → hardcoded default) means a revert can't silently fail to produce _some_ valid god type even if the ideal source of truth is missing.

## Files Required

- [[NetworkHandler]] — `broadcastMorph()`, called after every successful transform/revert.
- `AgentConfigLoader.java` (not yet read) — `getGodTypeForName()`, the fallback lookup in `removeTransform()`.

## Files Used In

- [[ServerGodAbilityExecutor]] — the confirmed, sole trigger point for `applyTransform()`/`removeTransform()`. **As of 2026-07-16, confirmed that the AI-driven path reaching that trigger point is itself broken further upstream** — see [[GodEntityManager]] and [[god-abilities]] for the full account. This class and [[ServerGodAbilityExecutor]] are both correctly written for a call that currently never arrives from an AI agent's own decisions.