---
type: file
status: ingested
---
# actuators.py

💡 **Role**: the command-out side matching [[wiki/codebase/files/vision|vision.py]]'s sensor-in — one factory (`ActuatorManager`) picking between a Minecraft TCP client and an Isaac Sim articulation driver. [[perception-and-actuation]] already documents the exact TCP wire format byte-for-byte and the flags-byte misalignment fix; this page adds a second fixed bug the topic page doesn't mention (a WebSocket self-loop), and the Isaac Sim DOF auto-detection machinery in full.

## Imports

`socket`, `threading`, `time`, `json`, `struct`, `logging`, `typing`, `numpy`, `asyncio`, `websockets` — external/stdlib. Isaac Sim modules (`isaacsim.core.utils.types.ArticulationAction`, `isaacsim.robot.wheeled_robots.controllers.differential_controller.DifferentialController`, `isaacsim.core.utils.numpy.rotations`) are all soft-imported with availability flags. No internal project imports.

## Module-level functions

- `build_inv_action(slot, button, click_type)` / `build_screen_action(command)` — construct the `"inv:SLOT,BUTTON,CLICK_TYPE"` / `"screen:COMMAND"` special-action strings [[perception-and-actuation]] documents; `INV_CLICK_*` constants mirror `net.minecraft.world.inventory.ClickType`'s ordinals exactly.
- `_infer_drive_mode(kp, kd)` — classifies an Isaac Sim joint as position/velocity/effort-driven purely from its gain values (both near zero → effort; only `kp` near zero → velocity; otherwise position).
- `_match_dofs(dof_names)` — auto-maps an unknown articulation's joint names to `move_forward`/`move_strafe`/`yaw_delta`/`pitch_delta`/`arm_position` by substring pattern matching against a priority-ordered table (`_ACTION_PATTERNS`), claiming each joint index for at most one action key.

## Classes

- **`BaseActuatorBackend`** — `apply_action()`/`get_observation()`/`close()`, the interface every backend implements.
- **`_WebSocketEventLoop`** — a process-wide singleton `asyncio` loop running on its own background thread, so synchronous callers elsewhere in the codebase can `submit()`/`submit_and_wait()` a coroutine without needing to be inside an event loop themselves.
- **`_TCPConnectionPool`** — one `ForgeIPCClient` instance per `(host, port)`, class-level cache — several agents on the same machine sharing a port reuse the same underlying socket rather than each opening their own.
- **`MinecraftClient`** — the actual `ActuatorManager` backend for `backend='minecraft'`. **Not on the topic page**: its own docstring documents a fixed bug (B-05) — an earlier version opened an _outbound_ WebSocket from Python back to Python's own FastAPI server (`ws_port == _backend_port`), a pure self-loop that never reached Java at all. Actions now travel exclusively via TCP (`ForgeIPCClient`); the real Python→Java WebSocket action path is a _response_ sent back over the same connection Java opened (`AgentConnectionHandler.send_action()` inside `handle_agent_websocket()`, in `communication_protocol.py`), not an outbound connection from here. `connect_websocket()` is kept as a no-op purely so existing call sites that still invoke it don't break.
- **`ForgeIPCClient`** — the real TCP sender. `_maintain()` runs a background reconnect loop (1s retry) independent of `send_action()`, so a send during a connection gap simply fails fast rather than blocking. `send_action()` packs the exact frame [[perception-and-actuation]] documents (own docstring cites the same two fixes that page's discrepancy note covers: the single-flags-byte fix and the previously-missing hotbar/sprint/god-ability fields).
- **`MinecraftWebSocketClient`** — a _separate_, independent WS client (not the one B-05 removed) with its own `_build_frame()` — same conceptual frame as `ForgeIPCClient.send_action()`, but with a distinct binary layout: a `MAGIC`+`FRAME_ACTION` header, clipped/scaled movement values (`yaw_delta` scaled ×2 clipped to ±180°, `pitch_delta` scaled ×1.2 clipped to ±90° — different scaling from the raw TCP path), and reconnect-on-demand (`send_action()`/`send_chat()` both kick off `connect()` as a fire-and-forget task if not currently connected, then simply drop that call rather than blocking on the reconnect).
- **`_DOFProfile`** (`__slots__`) — one Isaac Sim joint's name, index, drive mode, limits, and revolute flag.
- **`ActuatorAdapterIsaacSim`** — see [[perception-and-actuation]] for the concept. Mechanically: `_do_inspect()` reads every joint's gains/limits/types with a fallback chain at each step for Isaac Sim 5.0 API renames (`get_gains()`→`get_dof_gains()`, `get_dof_position_limits()`→`get_joint_limits()`, tolerating a missing `get_dof_types()` entirely by defaulting every joint to revolute), then runs `_match_dofs()` (overridable via a `joint_overrides` constructor argument) to build the action-key→joint-index mapping. If exactly two joints claim `move_forward` and either is named with "left"/"right", it assumes a differential-drive wheeled robot and builds a `DifferentialController` — `apply_action()` then routes through that controller's own `forward()` instead of per-joint velocity commands. Otherwise, `_apply_robot()` clips each mapped action value to that joint's own limits and dispatches by drive mode (position/velocity/effort), one `ArticulationAction` call per mode with all its joints batched together rather than one call per joint. `_apply_camera()` tracks yaw/pitch as running state (not absolute) and either sets a real camera prim's orientation (if `rot_utils` is available) or calls back through a provided `set_camera_ori` callback.
- **`ActuatorManager`** — `_BACKENDS = {"minecraft": MinecraftClient, "isaac": ActuatorAdapterIsaacSim}`; the constructor's `backend` string is the only thing selecting between an entirely different reality layer.

No async methods as class methods — `ForgeIPCClient`/`MinecraftWebSocketClient`'s async work runs through the module-level `_WebSocketEventLoop` singleton and `MinecraftWebSocketClient`'s own `async def connect()`/`send_action()`/`send_chat()`/`close()`, called via `asyncio.ensure_future()` or awaited directly by a caller that's already inside the shared loop.

## Problems (faced by traditional AI systems / LLMs)

Any system driving multiple, architecturally different targets (a game client over a network socket, an articulated robot body in a physics simulator) through one abstraction risks the abstraction leaking — either the interface has to be watered down to the lowest common denominator, or callers end up branching on which backend they're actually talking to. A narrower, robotics-specific problem: code written against one specific robot's joint layout doesn't transfer to a different articulation without a rewrite, and simulator SDKs renaming their own APIs across versions (as Isaac Sim's did) breaks integration code that assumed a fixed method name.

## Solutions

`BaseActuatorBackend`'s three-method interface is narrow enough that both a network client and a physics-sim driver can honestly implement it without stretching either one — `ActuatorManager` callers never need to know which backend is active. The DOF auto-detection in `ActuatorAdapterIsaacSim` solves the second problem generically: rather than hardcoding joint names for one specific robot, it pattern-matches against common naming conventions and falls back gracefully (unmapped joints simply don't receive commands), and the Isaac Sim 5.0 API fallback chains mean the same code keeps working across at least two versions of the underlying SDK's method names.

## Files Required

None from within this codebase — this file is a leaf dependency other modules import, not the other way around.

## Files Used In

- `agent.py` — constructs `ActuatorManager` with the appropriate backend string (see [[agent]] and [[agent-runtime]]).
- `communication_protocol.py` — `AgentConnectionHandler.send_action()` is the real Python→Java WebSocket action path `MinecraftClient`'s docstring points to; not yet given its own file-level page.
- [[human-controller-debug]] — `_pack_action` in `human_controller_server.py` builds a wire-compatible frame independently, matching the same flags-byte layout documented here.
- [[isaac-sim-integration]] — a separate, similarly-named `IsaacSimActuator` exists there; whether it's the same bridge, a different layer, or genuine duplication is still an open question from that page, not resolved by reading this file.