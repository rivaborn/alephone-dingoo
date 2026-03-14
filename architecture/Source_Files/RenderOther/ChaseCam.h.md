# Source_Files/RenderOther/ChaseCam.h

## File Purpose
Defines the interface for a third-person chase camera system in the Aleph One game engine (Marathon). Provides configuration, state management, and per-frame update logic for a Halo-like camera that follows the player with configurable physics-based positioning and damping.

## Core Responsibilities
- Define chase camera configuration data structure with offset and physics parameters
- Provide state query and control functions (active/inactive toggles)
- Manage camera initialization and reset on level transitions
- Update camera position each frame with spring/damping physics
- Calculate final camera position, orientation (yaw/pitch), and world polygon for rendering
- Support optional camera-switching based on horizontal offset

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `ChaseCamData` | struct | Holds configuration: behind/upward/rightward offsets (shorts), flags, damping/spring constants (floats), and opacity |

## Global / File-Static State
None (state management is delegated to implementation file).

## Key Functions / Methods

### ChaseCam_CanExist
- Signature: `bool ChaseCam_CanExist()`
- Purpose: Determine if chase cam can possibly activate; used to avoid loading player sprites if not needed
- Inputs: None
- Outputs/Return: `true` if can exist, `false` otherwise
- Side effects: None
- Notes: Optimization check; prevents unnecessary resource loading

### ChaseCam_IsActive / ChaseCam_SetActive
- Signature: `bool ChaseCam_IsActive()` / `bool ChaseCam_SetActive(bool NewState)`
- Purpose: Query or modify active state
- Outputs/Return: Current/new state
- Side effects: SetActive modifies global chase cam state

### ChaseCam_Initialize
- Signature: `bool ChaseCam_Initialize()`
- Purpose: Set up chase cam for a new game session
- Outputs/Return: Success status
- Side effects: Allocates/initializes camera state

### ChaseCam_Reset
- Signature: `bool ChaseCam_Reset()`
- Purpose: Reset camera when entering a level, reviving, or teleporting
- Side effects: Resets internal position/velocity state

### ChaseCam_Update
- Signature: `bool ChaseCam_Update()`
- Purpose: Update camera position each game tick using spring/damping physics
- Side effects: Modifies internal camera physics state
- Notes: Called once per frame to maintain correct physics; returns false if inactive

### ChaseCam_SwitchSides
- Signature: `bool ChaseCam_SwitchSides()`
- Purpose: Toggle camera to opposite side when horizontally offset
- Side effects: Modifies internal camera offset state

### ChaseCam_GetPosition
- Signature: `bool ChaseCam_GetPosition(world_point3d &position, short &polygon_index, angle &yaw, angle &pitch)`
- Purpose: Retrieve final camera position, world polygon, and orientation for rendering
- Inputs: Reference outputs for position, polygon, yaw, pitch
- Outputs/Return: `true` if active (outputs valid); `false` if inactive (outputs unchanged)
- Notes: All parameters are out-references; only modified if camera is active

## Control Flow Notes
Typical frame lifecycle: **Initialize** (game start) ΓåÆ **Update** (each tick) ΓåÆ **GetPosition** (for rendering). **Reset** is called on level transitions or teleport events. State can be toggled via **SetActive**; **SwitchSides** is called dynamically based on gameplay.

## External Dependencies
- `world.h` ΓÇö provides `world_point3d`, `angle` typedef, and world coordinate system definitions
