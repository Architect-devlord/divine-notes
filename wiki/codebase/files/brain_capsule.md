---
type: file
status: ingested
---
# brain_capsule.py

💡 **Role**: the actual `BrainCapsule` class — the single-file save format every `agent.save()`/`load()` call reads and writes. [[memory-and-braincapsule]] already documents its purpose and contents at the narrative level; this page is the mechanical catalog.

## Imports

`json`, `pickle`, `time`, `pathlib.Path`, `typing`, `dataclasses` — all external/stdlib. No internal project imports — this file is deliberately dependency-free so it can serialize/deserialize state from many different classes without importing any of them.

## Classes

### `BrainCapsule` (dataclass)

Fields: `agent_id`, `agent_type`, `personality_state`, `emotion_state`, `memory_snapshot`, `model_state` (a dict of raw `state_dict()`s from whichever neural components are attached — world model, policy, continual learner, and so on), `language_summary` (the `vocabulary_size`/`pattern_count` keys [[wiki/codebase/files/brain_language|brain_language.py]]'s `state_dict()` fix specifically added, since a separate summary reader expects those exact names), `created_at`/`updated_at` timestamps, `version`.

**Methods:**

- `save(path)` — writes two things: a human-readable `.json` sidecar (personality/emotion/language summary — anything JSON-safe) and the actual `.pcap` file itself via `pickle` (everything, including raw tensors/arrays the JSON sidecar can't represent). The sidecar is explicitly for quick inspection without unpickling, not a substitute for the real capsule.
- `load(path)` _(classmethod)_ — unpickles the `.pcap` file directly; doesn't read the JSON sidecar at all — that file exists purely for humans/tooling to peek at, not as a real data path.
- `to_dict()` / `from_dict()` — the JSON-safe subset used by the sidecar file, not a full round-trip (raw tensor data is dropped).
- `get_size_mb()` — file size of the `.pcap` on disk, for status reporting.

No async methods.

## Problems (faced by traditional AI systems / LLMs)

A save format built for a system with many independent, evolving neural components (a world model, a policy, a continual learner, each iterating on its own schedule) risks either a rigid, versioned binary format that breaks on every component change, or losing the ability to inspect what's actually saved without fully deserializing everything (including large tensors) just to check simple metadata.

## Solutions

Splitting `model_state` into a dict keyed by component name, rather than a fixed set of named fields, means adding or removing a neural component doesn't require changing this class at all — only whatever component's own `state_dict()`/`load_state_dict()` needs updating. The JSON sidecar solves the inspection problem: `updated_at`, `agent_type`, and the language summary are all readable with a text editor, without ever unpickling the (potentially large) real capsule.

## Files Required

None — deliberately dependency-free.

## Files Used In

- `agent.py` — `NPCAgent.save()`/`load()` construct and read a `BrainCapsule` (see [[agent]] and [[agent-runtime]]).
- Every component with its own `state_dict()`/`load_state_dict()` — `world_model.py`, `continual_learner.py`, `brain_language.py`, and others — contributes to and reads from `model_state` (see [[memory-and-braincapsule]] for the full list).