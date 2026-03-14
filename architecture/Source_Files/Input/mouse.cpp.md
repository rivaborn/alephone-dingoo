# Source_Files/Input/mouse.cpp

## File Purpose
Provides mouse input handling for the Marathon game engine on Mac (Carbon) platforms. Captures mouse movement and button events, converts them to game action flags and view delta values (yaw, pitch, velocity), and applies user-preference settings like sensitivity and acceleration.

## Core Responsibilities
- Initialize and teardown mouse input event handling via Carbon Event Manager
- Capture raw mouse position and button events from Carbon event stream
- Calculate frame-by-frame mouse deltas and convert to game action values (yaw, pitch, velocity)
- Apply input preferences (mouse sensitivity, Y-axis inversion, acceleration curves) to raw deltas
- Map mouse button presses to game triggers (left/right weapon fire) and scroll wheel to weapon cycling
- Handle mouse warping (centering) to enable relative-motion tracking
- Provide thread-safe communication between event handler (main thread) and input sampling (separate thread via critical regions)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `Point` | struct (Mac QuickDraw) | 2D position; x/y fields for mouse coordinates |
| `EventHandlerRef` / `EventHandlerUPP` | typedef (Carbon) | Reference and callback for event handlers |
| `MPCriticalRegionID` | typedef (Mac) | Thread synchronization primitive for shared mouse deltas |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `snapshot_delta_yaw` | `_fixed` | static | Current frame's yaw delta (left/right view rotation) |
| `snapshot_delta_pitch` | `_fixed` | static | Current frame's pitch delta (up/down view angle) |
| `snapshot_delta_velocity` | `_fixed` | static | Current frame's forward/backward movement delta |
| `snapshot_button_state` | `bool[MAX_BUTTONS]` | static | Pressed/released state for each mouse button |
| `snapshot_delta_scrollwheel` | `int` | static | Scroll wheel direction/count this frame |
| `CE_MouseLock` | `MPCriticalRegionID` | static | Lock protecting access to shared deltas |
| `_CE_delta_x`, `_CE_delta_y` | `int` | static | Raw mouse position deltas from events |
| `_CE_delta_scrollwheel` | `int` | static | Raw scroll wheel delta from events |
| `_CEMouseTrackerUPP`, `_CEMouseTracker` | pointers | static | Carbon event handler reference/UPP |
| `PrevPosition` | `Point` | static | Previous position for Classic fallback mode |
| `CGGetLastMouseDelta_Ptr`, `CGWarpMouseCursorPosition_Ptr` | function pointers | static | Dynamically loaded CoreGraphics functions |

## Key Functions / Methods

### enter_mouse
- **Signature:** `void enter_mouse(short type)`
- **Purpose:** Initialize mouse input subsystem; install Carbon event handler and create thread synchronization lock.
- **Inputs:** `type` ΓÇö mouse input mode (_mouse_yaw_pitch or _mouse_yaw_velocity); currently unused but reserved.
- **Outputs/Return:** None.
- **Side effects (global state, I/O, alloc):** Allocates critical region (MPCreateCriticalRegion), installs application-level Carbon event handler, initializes all snapshot variables to zero/false.
- **Calls (direct calls visible in this file):** `NewEventHandlerUPP`, `InstallApplicationEventHandler`, `MPCreateCriticalRegion`, `GetGlobalMouse` (fallback), `LoadCGMouseFunctions`.
- **Notes:** If critical region allocation fails (possible in Classic mode), falls back to reading global mouse position directly. Installs handlers for kEventMouseDown, kEventMouseUp, kEventMouseWheelMoved, kEventMouseMoved, kEventMouseDragged.

### test_mouse
- **Signature:** `void test_mouse(short type, uint32 *action_flags, _fixed *delta_yaw, _fixed *delta_pitch, _fixed *delta_velocity)`
- **Purpose:** Apply current frame's mouse snapshot to player action flags and return movement deltas.
- **Inputs:** `type` ΓÇö input mode (unused); `action_flags` ΓÇö pointer to bitmask to modify; `delta_yaw`, `delta_pitch`, `delta_velocity` ΓÇö pointers to receive movement deltas.
- **Outputs/Return:** Updates action_flags with trigger/weapon-cycle bits; fills delta pointers.
- **Side effects (global state, I/O, alloc):** Modifies input_preferences flags and clears snapshot_delta_scrollwheel.
- **Calls (direct calls visible in this file):** None visible (uses global snapshot variables).
- **Notes:** Maps mouse button states to _left_trigger_state, _right_trigger_state via input_preferences->mouse_button_actions. Scroll wheel sets _cycle_weapons_forward/backward flags.

### exit_mouse
- **Signature:** `void exit_mouse(short type)`
- **Purpose:** Shut down mouse input; remove event handler and free resources.
- **Inputs:** `type` ΓÇö input mode (unused).
- **Outputs/Return:** None.
- **Side effects (global state, I/O, alloc):** Removes event handler, disposes UPP, deletes critical region.
- **Calls (direct calls visible in this file):** `RemoveEventHandler`, `DisposeEventHandlerUPP`, `MPDeleteCriticalRegion`.
- **Notes:** Clears handler and lock pointers to NULL to prevent use-after-free.

### mouse_idle
- **Signature:** `void mouse_idle(short type)`
- **Purpose:** Poll current mouse position, calculate frame deltas, apply sensitivity/acceleration, update snapshot variables. Called once per game tick.
- **Inputs:** `type` ΓÇö mouse mode; selects whether vy becomes delta_pitch or delta_velocity.
- **Outputs/Return:** None; updates global snapshots.
- **Side effects (global state, I/O, alloc):** Updates snapshot_delta_yaw/pitch/velocity, recalculates button state, warps mouse to center position, reads/resets scrollwheel delta from critical region.
- **Calls (direct calls visible in this file):** `get_mouse_location`, `set_mouse_location`, `TickCount`, `Button`, `MPEnterCriticalRegion`, `MPExitCriticalRegion`.
- **Notes:** Calculates velocities in fixed-point; clamps to ┬▒FIXED_ONE/2, optionally applies non-linear acceleration curve (squaring) if input_preferences->mouse_acceleration is true. Respects input_preferences->sens_horizontal, sens_vertical, and _inputmod_invert_mouse flag. Uses static first_run and last_tick_count to handle init and frame-delta calculation.

### get_mouse_location
- **Signature:** `static void get_mouse_location(Point *where)`
- **Purpose:** Retrieve current mouse position (relative to screen center) from critical-region deltas or fallback GetGlobalMouse.
- **Inputs:** `where` ΓÇö pointer to Point to fill.
- **Outputs/Return:** Fills where->h, where->v with CENTER_MOUSE_X/Y + accumulated deltas.
- **Side effects (global state, I/O, alloc):** Resets _CE_delta_x/y to zero after reading (if critical region succeeds); updates PrevPosition if using fallback.
- **Calls (direct calls visible in this file):** `MPEnterCriticalRegion`, `MPExitCriticalRegion`, `GetGlobalMouse`.
- **Notes:** Returns position centered at (320, 240). Fallback path tracks delta from previous frame for compatibility.

### set_mouse_location
- **Signature:** `static void set_mouse_location(Point where)`
- **Purpose:** Warp (move) the mouse cursor to a screen location.
- **Inputs:** `where` ΓÇö target position.
- **Outputs/Return:** None.
- **Side effects (global state, I/O, alloc):** Issues CoreGraphics warp command.
- **Calls (direct calls visible in this file):** `CGWarpMouseCursorPosition` or `CGWarpMouseCursorPosition_Ptr`.
- **Notes:** On __MACH__ uses direct call; on other platforms uses function pointer (loaded via LoadCGMouseFunctions). Coordinates are screen-global.

### CEvtHandleApplicationMouseEvents
- **Signature:** `pascal OSStatus CEvtHandleApplicationMouseEvents(EventHandlerCallRef nextHandler, EventRef theEvent, void* userData)`
- **Purpose:** Carbon event handler callback; receives mouse events and updates shared delta/button state variables. Runs in main thread.
- **Inputs:** `nextHandler` ΓÇö next handler in chain; `theEvent` ΓÇö Carbon event; `userData` ΓÇö context (unused).
- **Outputs/Return:** OSStatus (noErr or eventNotHandledErr).
- **Side effects (global state, I/O, alloc):** Updates _CE_delta_x/y/scrollwheel via critical region; updates snapshot_button_state directly; warps cursor to center.
- **Calls (direct calls visible in this file):** `GetEventKind`, `GetEventClass`, `CallNextEventHandler`, `get_game_state`, `process_screen_click`, `FrontWindow`, `ConvertEventRefToEventRecord`, `CGGetLastMouseDelta`, `GetEventParameter`, `MPEnterCriticalRegion`, `MPExitCriticalRegion`, `CGWarpMouseCursorPosition`.
- **Notes:** Handles kEventMouseMoved, kEventMouseDragged (accumulates CoreGraphics deltas), kEventMouseDown/Up (updates button state), kEventMouseWheelMoved (accumulates scroll). Returns early if game is not in progress or displaying chapter heading.

### LoadCGMouseFunctions
- **Signature:** `void LoadCGMouseFunctions()`
- **Purpose:** Dynamically load CoreGraphics function pointers (for non-__MACH__ platforms).
- **Inputs:** None.
- **Outputs/Return:** None; sets global function pointers.
- **Side effects (global state, I/O, alloc):** Sets CGWarpMouseCursorPosition_Ptr and CGGetLastMouseDelta_Ptr via GetSystemFunctionPointer.
- **Calls (direct calls visible in this file):** `GetSystemFunctionPointer`.
- **Notes:** Only called on non-__MACH__ systems; allows fallback on older Mac OS versions without these CoreGraphics APIs.

**Helper function notes:**
- `get_absolute_pitch_range()` ΓÇö declared in player.h; returns valid pitch range for absolute positioning mode.

## Control Flow Notes
Mouse input integrates into the game's per-frame input pipeline:
1. **Initialization** (enter_mouse): Install event handler, create synchronization lock, zero snapshot variables.
2. **Event loop** (main thread): Carbon event handler (CEvtHandleApplicationMouseEvents) receives mouse events, updates shared _CE_delta_* variables under lock.
3. **Game tick** (input thread or same thread):
   - `mouse_idle()` called once per frame: reads position deltas from critical region, warps cursor to center, applies sensitivity/acceleration, stores in snapshot_* variables.
   - `test_mouse()` called: reads snapshot_* variables, maps to action_flags and output deltas.
4. **Shutdown** (exit_mouse): Remove event handler, free lock and UPP.

The snapshot variables decouple the asynchronous event handler from the synchronous game tick, allowing frame-coherent input sampling.

## External Dependencies
- **Marathon engine includes:**
  - world.h ΓÇö integer_to_fixed conversion, world coordinate types.
  - map.h ΓÇö player-related structures, game state queries.
  - player.h ΓÇö get_absolute_pitch_range() for pitch clamping.
  - shell.h ΓÇö interface declarations, get_game_state(), process_screen_click().
  - interface.h ΓÇö FrontWindow(), screen_window (Carbon).
  - preferences.h ΓÇö input_preferences global (sensitivity, acceleration, button mappings, modifier flags).
  - Logging.h ΓÇö dprintf() macro (conditional debug logging).

- **Mac/Carbon APIs:**
  - ApplicationServices.h (TARGET_API_MAC_CARBON) ΓÇö Carbon Event Manager, Point type, critical regions, dynamic function loading.
  - CoreGraphics (CGDirectDisplay.h) ΓÇö CGGetLastMouseDelta, CGWarpMouseCursorPosition (function pointers).
  - macintosh_cseries.h ΓÇö fixed-point types (_fixed), utility macros (PIN, TEST_FLAG).

- **External symbols defined elsewhere:**
  - `input_preferences`, `dynamic_world` ΓÇö globals from preferences.cpp and map.cpp.
  - `process_screen_click()`, `get_game_state()`, `FrontWindow()`, `GetGlobalMouse()`, `Button()` ΓÇö declared in shell.h or Carbon headers.
