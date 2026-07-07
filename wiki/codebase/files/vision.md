---
type: file
status: ingested
---

# vision.py

💡 **Role**: the agent's own eyes — capture, feature extraction, and a self-grown visual vocabulary with zero pretrained labels. [[perception-and-actuation]] already covers the four capture backends, the CNN-not-pretrained commitment, and the online-vocabulary concept in real depth; this page adds the mechanical catalog, the three separate monkeypatch points that actually wire this into an agent, and a third instance of a stale-dimension finding already flagged on [[world-model]].

## Imports

`logging`, `threading`, `time`, `queue`, `abc` (`ABC`, `abstractmethod`), `dataclasses`, `typing`, `numpy` — external/stdlib. `torch`/`cv2` (OpenCV)/Isaac Sim modules are all soft-imported (module-level `try`/`except`, flags like `_TORCH`/`_CV2`/`_ISAAC` gate their use throughout — not shown in the structural grep since they're conditional). No internal project imports; `communication_protocol` is imported lazily, inside `_wire_minecraft_protocol()` only.

## Classes

- **`VisualFrame`** (dataclass) — one processed frame: raw pixels, CHW tensor, feature vector, assigned visual token, source, timestamp, optional depth. `to_memory_event()` and `to_proprio_vector(dim=32)` are the two serialization paths — the latter is what actually reaches [[world-model]]'s `WorldModelReplayBuffer`.
    
- **`_SimpleCNNExtractor`** (`nn.Module`) — 3-layer conv stack, randomly initialized, no pretrained weights by design (see [[perception-and-actuation]]).
    
- **`FeatureExtractor`** — wraps the CNN; `extract()` falls back to eight plain pixel statistics (per-channel mean/std, 10th/90th percentile) when torch isn't available at all, padded/truncated to `feature_dim`. `state_dict()`/`load_state_dict()` for brain-capsule persistence.
    
- **`OnlineVisualVocabulary`** — see [[perception-and-actuation]] for the concept. Mechanically: `observe()` does one step of online k-means (nearest-centre update, `lr=0.05`) and considers splitting off a new cluster only if the matched centre has absorbed at least `min_obs_to_split` (50) observations _and_ the trigger point is more than 1.5× the vocabulary's own mean intra-cluster distance away — both conditions, not either. `_mean_intra_dist()` samples up to 20 random centre pairs rather than computing the full pairwise distance matrix, an O(1)-ish approximation as cluster count grows. `most_similar()`, `assign_name()`/`name_of()` for language grounding, full `state_dict()`/`load_state_dict()`.
    
- **`CaptureBackend`** (ABC) — `read()`, `is_available()`, `close()`, `source_name()`.
    
- **`SyntheticBackend`** — deterministic animated gradient frames (sine/cosine waves parameterized by an internal tick counter), always available; the CI/no-hardware fallback.
    
- **`RobotCameraBackend`** — see [[perception-and-actuation]]; auto-detects device indices 0–3 if no explicit source is given.
    
- **`MinecraftVisionBackend`** — `push_frame()` is the actual injection point [[electron-frontend]]/`communication_protocol.py` call; thread-locked latest-frame buffer, not a queue — a new frame simply replaces the old one rather than accumulating.
    
- **`IsaacSimVisionBackend`** — `attach_camera()` binds a live camera prim after `world.reset()`; `read()` converts RGBA→RGB→BGR and resizes.
    
- **`ContextDetector`** — `detect()` is the priority-ordered backend list builder: explicit source (exclusive) → Minecraft (always included, becomes available once the mod connects) → Isaac Sim (if the SDK is importable) → physical camera (if OpenCV finds one) → synthetic (always last, guaranteed available).
    
- **`SelfTrainedDepthEstimator`** — not a network at all despite the name; a gradient-magnitude heuristic (`10.0 * (1 - normalized_gradient_magnitude)`), explicitly a placeholder ("we never rely on a pretrained network") rather than a trained depth head.
    
- **`VisionAdapter`** — the public interface. `observe()` is the primary entry point: processes a frame (or captures a fresh one), stores a memory event, forwards a proprio vector to [[world-model]]'s replay buffer, and appends a self-generated "I see visual pattern X" thought if the agent has a `thoughts` list. `start()`/`stop()`/`_capture_loop()` run this on a background thread at a target FPS (default 15) independent of the cognitive loop's own tick rate. `push_minecraft_frame()`/`attach_isaac_camera()` are the two context-specific integration points. `get_frame()`/`preprocess()` are kept as legacy-compatible aliases over the same pipeline.
    
    **Same stale-dimension pattern flagged on [[world-model]] and [[wiki/codebase/files/world_model|world_model.py]]'s file-level page**: `observe()`'s fallback when `agent.last_action` isn't set is `np.zeros(11, ...)` — a third call site (after `test_world_model()`'s dummy batch and `_build_observation_from_context()`'s fallback) still using the pre-fix width instead of the corrected 13.
    

## Functions

- `add_vision_to_agent(agent, ...)` — the single call site that builds a `VisionAdapter` and wires it in via three separate monkeypatches, not one: `_patch_agent_observe()` (replaces `agent.observe` itself), `_wire_minecraft_protocol()` (hooks `communication_protocol._on_visual_frame`, chaining to whatever handler was already registered rather than replacing it), and `_patch_cognitive_loop()` (wraps `cognitive_loop._perceive` to inject `visual_token`/`visual_token_name`/novelty into the perception dict, taking the max of existing and visual novelty rather than overwriting it).
- `_patch_agent_observe(agent)` / `_wire_minecraft_protocol(agent)` / `_patch_cognitive_loop(agent)` — see above; all three are genuine function-replacement monkeypatches, the same pattern [[continual_learner]] uses for `agent.continual_learner` (attribute assignment) taken one step further (actual method replacement).

No async methods anywhere in this file — the background capture loop is a plain `threading.Thread`, not `asyncio`.

## Problems (faced by traditional AI systems / LLMs)

Vision systems built on pretrained backbones (ImageNet-style categories) impose the designer's taxonomy on the agent before it has ever seen anything — a fixed vocabulary of "what things are" that can't grow to match what a specific agent, in a specific world, actually encounters. A second, more infrastructural problem: wiring a new sensory system into an already-running agent typically means either rewriting every call site that used the old sensory path, or accepting that the new system only partially replaces the old one.

## Solutions

Online k-means with no pretrained labels (`OnlineVisualVocabulary`) means the vocabulary's size and content are entirely a function of what this specific agent has actually seen — two agents in different environments end up with different, equally valid visual "languages." For the wiring problem: monkeypatching the exact methods and hooks other code already calls (`agent.observe`, `cognitive_loop._perceive`, the protocol's frame callback) means every existing call site picks up the new vision pipeline automatically, with no call site needing to know vision was ever added.

## Files Required

None from within `ai_core/` at module level — `communication_protocol.py` is imported lazily inside `_wire_minecraft_protocol()` only, and even that import is wrapped in a `try`/`except` that silently skips the hook if it fails.

## Files Used In

- `agent.py` — `add_vision_to_agent()` is called from `NPCAgent.__init__` (see [[agent]] and [[agent-runtime]]); patches `agent.observe` directly.
- `cognitive_loop.py` — `_perceive()` gets wrapped to include visual token/novelty (see [[cognitive_loop]]).
- `world_model.py` — every `observe()` call forwards a proprio vector into `agent.world_model_buffer` (see [[wiki/codebase/files/world_model|world_model.py]] and [[world-model]]).
- `communication_protocol.py` — `push_minecraft_frame()` is the actual entry point for Minecraft vision packets; not yet given its own file-level page.