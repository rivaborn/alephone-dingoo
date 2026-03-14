# Source_Files/RenderOther/ChaseCam.cpp

## File Purpose
Implements third-person chase camera functionality that follows the player. The camera supports configurable offset, spring physics for smooth motion, wall collision handling, and network restrictions. Designed to provide Halo-like behind-the-player viewing.

## Core Responsibilities
- Manage chase camera activation lifecycle (enable/disable, check prerequisites)
- Maintain camera position state with history for physics calculations
- Calculate camera position based on player location with configurable offsets (behind, upward, rightward)
- Apply spring-damper physics for smooth camera movement
- Handle wall collisions using polygon/line tracing
- Provide camera state reset for level transitions and teleports
- Support horizontal camera offset switching (left/right sides)

## Key Types / Data Structures
None defined in this file. Uses:
| Name | Kind | Purpose |
|------|------|---------|
| world_point3d | struct | Camera position (x, y, z) |
| polygon_data | struct | Map polygon geometry (defined in map.h) |
| line_data | struct | Map line segment (defined in map.h) |
| ChaseCamData | struct | Configuration (offset, damping, spring, opacity) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| _ChaseCam_IsActive | bool | static | Active/inactive toggle |
| _ChaseCam_IsReset | bool | static | Tracks if camera position was reset last tick |
| CC_Position | world_point3d | static | Current camera world position |
| CC_Position_1 | world_point3d | static | Previous frame position (for spring physics) |
| CC_Position_2 | world_point3d | static | Two frames ago (for spring physics) |
| CC_Polygon | short | static | Polygon index containing camera |
| CC_Yaw | angle | static | Horizontal camera rotation |
| CC_Pitch | angle | static | Vertical camera rotation |

## Key Functions / Methods

### ChaseCam_CanExist
- Signature: `bool ChaseCam_CanExist()`
- Purpose: Check if chase camera feature is enabled in configuration
- Inputs: None (reads GetChaseCamData().Flags)
- Outputs/Return: `true` if camera can activate; `false` if disabled
- Side effects: None
- Calls: `GetChaseCamData()`
- Notes: Called as guard condition in most other functions; used to avoid loading player sprite assets unnecessarily

### ChaseCam_IsActive
- Signature: `bool ChaseCam_IsActive()`
- Purpose: Query whether camera is currently active
- Inputs: None (reads _ChaseCam_IsActive)
- Outputs/Return: `true` if active; `false` if disabled or cannot exist
- Side effects: None
- Calls: `ChaseCam_CanExist()`, `NetAllowBehindview()` (conditional on DISABLE_NETWORKING)
- Notes: Network games can restrict behind-view (cosmetic restriction on deathmatch)

### ChaseCam_SetActive
- Signature: `bool ChaseCam_SetActive(bool NewState)`
- Purpose: Enable or disable the camera
- Inputs: `NewState` ΓÇô target state
- Outputs/Return: New active state; `false` if cannot activate
- Side effects: Updates `_ChaseCam_IsActive`
- Calls: `ChaseCam_CanExist()`, `NetAllowBehindview()`
- Notes: Subject to network restrictions; caller should verify return value

### ChaseCam_Initialize
- Signature: `bool ChaseCam_Initialize()`
- Purpose: Set up camera for a new game or level
- Inputs: None (reads GetChaseCamData().Flags)
- Outputs/Return: `true` if initialized successfully
- Side effects: Calls `ChaseCam_Reset()` and sets initial active state from flags
- Calls: `ChaseCam_CanExist()`, `ChaseCam_Reset()`, `ChaseCam_SetActive()`, `GetChaseCamData()`
- Notes: Invoked at game/level start

### ChaseCam_Reset
- Signature: `bool ChaseCam_Reset()`
- Purpose: Reset camera position (after teleport, level transition, or respawn)
- Inputs: None
- Outputs/Return: `true` if reset; `false` if not active or cannot exist
- Side effects: Sets `_ChaseCam_IsReset = true` (signals `ChaseCam_Update()` to reinitialize position history)
- Calls: `ChaseCam_CanExist()`, `ChaseCam_IsActive()`
- Notes: Prevents stale position data from affecting spring physics after discontinuous movement

### ChaseCam_SwitchSides
- Signature: `bool ChaseCam_SwitchSides()`
- Purpose: Negate the rightward offset (flip camera from left to right side or vice versa)
- Inputs: None
- Outputs/Return: `true` if switched; `false` if not active
- Side effects: Modifies `GetChaseCamData().Rightward`
- Calls: `ChaseCam_CanExist()`, `ChaseCam_IsActive()`, `GetChaseCamData()`
- Notes: Allows player to shift camera to opposite side dynamically

### ChaseCam_GetPosition
- Signature: `bool ChaseCam_GetPosition(world_point3d &position, short &polygon_index, angle &yaw, angle &pitch)`
- Purpose: Query current camera position and orientation (called each frame for rendering)
- Inputs: References to output variables
- Outputs/Return: `true` if updated; `false` (outputs unchanged) if not active
- Side effects: None (outputs are only written if active)
- Calls: `ChaseCam_CanExist()`, `ChaseCam_IsActive()`
- Notes: Caller must check return value; does not alter game state

### CC_PosUpdate (static helper)
- Signature: `static int CC_PosUpdate(float Damping, float Spring, short x0, short x1, short x2)`
- Purpose: Apply spring-damper equation to smooth camera motion
- Inputs: Damping coefficient, spring stiffness, current position (x0), previous (x1), two-frames-ago (x2)
- Outputs/Return: New smoothed position (rounded to integer)
- Side effects: None
- Calls: None
- Notes: Physics: `x = x0 + 2*D*(x1-x0) - (D┬▓+S)*(x2-x0)` where D=damping, S=spring; reduces jitter while maintaining responsiveness

### ShootForTargetPoint (static helper)
- Signature: `static void ShootForTargetPoint(bool ThroughWalls, world_point3d& StartPosition, world_point3d& EndPosition, short& Polygon)`
- Purpose: Trace line from start to end position, clipping to map geometry (wall collision)
- Inputs: `ThroughWalls` ΓÇô if true, pass through walls and return last valid polygon; `StartPosition` ΓÇô ray origin; `EndPosition` ΓÇô target; `Polygon` ΓÇô starting polygon index
- Outputs/Return: Modifies `EndPosition` (clipped) and `Polygon` (final polygon)
- Side effects: None (except caller-passed references)
- Calls: `get_polygon_data()`, `find_floor_or_ceiling_intersection()`, `find_line_crossed_leaving_polygon()`, `get_line_data()`, `get_endpoint_data()`, `find_line_intersection()`, `find_adjacent_polygon()`
- Notes: Prevents camera from clipping through walls; loop-based polygon traversal handles arbitrary map complexity; handles wraparound edge cases (short integer overflow)

### ChaseCam_Update (main update)
- Signature: `bool ChaseCam_Update()`
- Purpose: Advance camera state for one game tick
- Inputs: None (reads `current_player`, `GetChaseCamData()`, `_ChaseCam_IsReset`)
- Outputs/Return: `true` if updated; `false` if not active
- Side effects: Updates `CC_Position`, `CC_Position_1`, `CC_Position_2`, `CC_Polygon`, `CC_Yaw`, `CC_Pitch`, `_ChaseCam_IsReset`
- Calls: `ChaseCam_CanExist()`, `ChaseCam_IsActive()`, `GetChaseCamData()`, `translate_point3d()`, `translate_point2d()`, `CC_PosUpdate()`, `ShootForTargetPoint()`
- Notes: Multi-phase logic: (1) if not reset, shift position history; (2) compute target position from player + offsets; (3) if not reset, optionally apply inertia with spring-damper; (4) clip to walls; (5) if was reset, initialize history and clear flag. Handles both smooth continuous motion and discrete resets.

## Control Flow Notes
**Lifecycle:**
- **Initialization** (start of game/level): `ChaseCam_Initialize()` is called once.
- **Per-frame update** (~30 ticks/sec): `ChaseCam_Update()` is called each tick, then `ChaseCam_GetPosition()` is called before rendering to retrieve current view parameters.
- **Events**: `ChaseCam_Reset()` on level transitions or teleports; `ChaseCam_SwitchSides()` on player input.

**Physics pipeline:**
1. Player position copied to camera reference position.
2. Target position calculated: offset behind (local Z-axis), upward (world Z), and to the side (perpendicular to facing).
3. If not reset: position history shifted, inertia applied via `CC_PosUpdate()`.
4. Wall collision clipping via `ShootForTargetPoint()`.
5. Result stored in persistent static state for rendering.

## External Dependencies
- **map.h**: `polygon_data`, `line_data`, `endpoint_data`, geometry query functions (`get_polygon_data()`, `find_line_crossed_leaving_polygon()`, `find_adjacent_polygon()`, `find_floor_or_ceiling_intersection()`, `find_line_intersection()`, `get_line_data()`, `get_endpoint_data()`)
- **player.h**: `current_player` (global player data); `world_location3d` field access
- **network.h**: `NetAllowBehindview()` (cosmetic restriction in network games)
- **world.h**: `world_point3d`, `world_point2d`, `angle`, `world_distance` types; `translate_point3d()`, `translate_point2d()` (vector transformation)
- **ChaseCam.h**: Configuration struct `ChaseCamData`, function declarations
- **cseries.h**: Base types and macros (`TEST_FLAG`)
