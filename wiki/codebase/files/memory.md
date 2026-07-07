---
type: file
status: ingested
---
# memory.py

💡 **Role**: three storage systems living in one file — general event memory, an optional ScyllaDB persistence layer beneath it, and the RL replay buffer, which is a genuinely separate thing despite the name similarity. [[memory-and-braincapsule]] already covers `UnifiedMemoryStore`'s and `EpisodicMemory`'s roles and `BrainCapsule`'s full contents in real depth; this page adds `ScyllaMemoryBackend`'s own mechanics (not detailed there at all) and two real, undocumented query-correctness fixes.

## Imports

`time.time`, `uuid`, `json`, `logging`, `numpy`, `typing`, `collections` (`deque`, `defaultdict`) — external/stdlib. `cassandra-driver` (`Cluster`, `TokenAwarePolicy`, `DCAwareRoundRobinPolicy`, `BatchStatement`, etc.) is soft-imported behind a `CASSANDRA_AVAILABLE` flag. No internal project imports.

## Classes

- **`ScyllaMemoryBackend`** — **not detailed on the topic page at all.** `__init__` tries a real connection first; on failure, if any contact point is `localhost`/`127.0.0.1`, calls `_try_start_local_scylla()` — a genuine auto-provisioning attempt (Docker first: check for an existing `dw-scylla` container, start it if stopped, create it fresh with `scylladb/scylla:5.4` and modest resource limits if it doesn't exist at all; `systemctl` as a Linux-only fallback) — waits 8 seconds, then retries the connection once before giving up and falling back to in-memory-only. `_ensure_keyspace()`/`_ensure_tables()` create a `NetworkTopologyStrategy` keyspace (replication factor 3) with four tables: the main time-bucketed `memories` table (`TimeWindowCompactionStrategy`, one-day windows — the write pattern this whole schema is optimized for), plus `memories_by_tag`, `memories_by_type`, and `memory_embeddings` as separate index tables. `save_event()` writes to all relevant tables in a single `BatchStatement` at `LOCAL_QUORUM`. `query_recent()` walks backward through up to 8 daily buckets accumulating results until `limit` is met.
    
    **Two real fixes, neither mentioned on the topic page**: `query_by_tags()` and `query_by_type()` both used to fetch matching `event_id`s from their respective index tables and then call `query_recent()` — which ignores tag/type filters entirely — making the index lookup a dead join that filtered nothing. Both are now fixed the same way: use the index query only to find the _oldest relevant timestamp_, then call `query_recent(since=that_timestamp)` and filter the (now correctly bounded) result set down to the actual matching IDs.
    
- **`UnifiedMemoryStore`** — see [[memory-and-braincapsule]] for the overall role. `remember()` merges caller-supplied tags with any tags already on the event dict (via `dict.fromkeys` to preserve order while deduping), maintains in-memory tag/type indexes alongside the deque, and persists to Scylla if attached. `recall()` delegates to Scylla only when the in-memory cache exceeds 1000 events _and_ a specific filter (tags/type/since) is given — otherwise it's a plain in-memory reverse scan.
    
    **`search()`, not on the topic page**: its own comment traces a real reported gap — this method used to only ever look at the capped in-memory deque, unlike `recall()`, which explicitly falls back to Scylla for large datasets. Since Scylla/Cassandra have no native substring search, the fix couldn't be a true delegated query the way `recall()`'s is; instead, when the in-memory pass doesn't fill the request, it widens the candidate pool via `query_recent()` (over-fetching 8× the remaining need) and applies the same substring filter locally — the same "delegate when the fast path runs out" shape as `recall()`, just implemented as widen-then-filter since Scylla can't filter by substring itself. `search_by_tag()` is a new method added for the same underlying reason, added specifically because `brain_language.py` tags cross-agent conversation history as `partner:{id}` — an exact-tag lookup sidesteps needing a substring match at all for that specific, common case.
    
    `semantic_search()` (cosine similarity over the embedding cache), `get_training_batch()` (exponentially-recency-weighted sampling, optionally tag-filtered), `novelty_score()` (`1/(1+matches)` against a 200-character text prefix), `get_stats()`, `close()`.
    
- **`EpisodicMemory`** — see [[memory-and-braincapsule]] for its role and the once-never-initialized bug already documented there. Mechanically: prioritized replay via `priorities ** alpha` as unnormalized sampling weights, importance-sampling weights computed as `(N * prob)^-1` and normalized by their own max (standard PER). `is_ready(min_size=1000)` is the gate a training loop checks before sampling at all.
    

No async methods anywhere in this file.

## Functions

None at module level beyond the classes — `Memory = UnifiedMemoryStore` at the very end is a plain alias, not a function, kept so any older `from ai_core.memory import Memory` import continues to work unchanged.

## Problems (faced by traditional AI systems / LLMs)

Any memory system built to scale past what fits comfortably in RAM faces a recurring shape of bug: a fast path (recent, in-memory) and a slow path (persisted, queryable by index) that are supposed to agree on results, where it's easy for a filter to be correctly _computed_ against the index but then silently discarded when the actual data-fetching step reuses a method that doesn't know about that filter. A related, narrower problem specific to this codebase's dual-store design: a database built for time-series and key-based lookups (Scylla/Cassandra) fundamentally can't do the substring search an in-memory `list` can, so a naive "just add a persistence layer" migration silently loses that capability for anything old enough to have aged out of the in-memory cache.

## Solutions

Both dead-join bugs get the same fix: use the index query to establish _bounds_ (a timestamp), then re-run the already-correct bounded query and filter locally to the exact matching set — rather than trying to make the index query itself return full results directly. The substring-search-over-Cassandra problem gets an honest, explicitly-documented compromise: widen the candidate pool via the one query type Scylla _can_ do efficiently (recency-bounded fetch), then filter substrings in Python — slower than a real search index, but correct, and scoped to only kick in once the in-memory fast path is actually insufficient.

## Files Required

None from within this codebase — this file has no internal project imports.

## Files Used In

- `agent.py` — constructs `UnifiedMemoryStore` and `EpisodicMemory` (see [[agent]] and [[agent-runtime]]); `BrainCapsule.memory_snapshot` and `model_state`'s transition buffer both come from these two classes on save (see [[memory-and-braincapsule]]).
- `brain_language.py` — tags cross-agent conversation history with `partner:{id}`, the specific use case `search_by_tag()` was added for.
- Nearly every file in this batch (`vision.py`, `audio_processors.py`, `cognitive_loop.py`, `brain_core.py`) calls `agent.memory.remember()` directly to log its own events — this file is a leaf dependency with very broad fan-in, not a narrow one.