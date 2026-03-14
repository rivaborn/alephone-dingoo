# Source_Files/Misc/PlayerImage_sdl.cpp

## File Purpose
Implements SDL-based 2D sprite rendering for player characters in Aleph One. Manages separate leg and torso sprites with dynamic color, animation frame, and weapon state selection, supporting lazy resolution of player appearance data with automatic retry on invalid states.

## Core Responsibilities
- Allocate and free SDL surfaces for player leg and torso sprites
- Resolve unspecified player state properties (view, frame, color, action, weapon) through randomization
- Validate resolved shape data against shape definition tables; retry with new random choices on failure
- Manage SDL surface and pixel buffer lifecycle (alloc/free)
- Blit composited leg and torso sprites to target surfaces at specified world coordinates
- Track collection load/unload requests via reference counting across active instances

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| PlayerImage | class | Player sprite state, drawing data, and lifecycle management (defined in header) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| sNumOutstandingObjects | int16 (static) | class | Reference count of active PlayerImage instances; used to mark/unmark shape collection in memory |
| bit_depth | short (extern) | global | Current display bit depth; 8-bit depth causes shape surface fetch to skip (fail gracefully) |

## Key Functions / Methods

### ~PlayerImage (destructor)
- **Signature:** `~PlayerImage()`
- **Purpose:** Clean up SDL surfaces and pixel buffers, decrement object counter, unload collections if no instances remain.
- **Inputs:** None (operates on member state)
- **Outputs/Return:** None
- **Side effects:** Frees `mLegsSurface`, `mTorsoSurface`, `mLegsData`, `mTorsoData`; calls `objectDestroyed()`.
- **Calls:** `SDL_FreeSurface()`, `free()`, `objectDestroyed()`
- **Notes:** Checks pointers for NULL before freeing; safe to call multiple times.

### setRandomFlatteringView
- **Signature:** `void setRandomFlatteringView()`
- **Purpose:** Pick a random cardinal view (0ΓÇô7) that avoids the "butt shot" angle range (views 3ΓÇô5).
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Modifies `mLegsView` and `mTorsoView` via `setView()`; marks both as dirty.
- **Calls:** `local_random()`, `setView()`
- **Notes:** Loops until a non-excluded view is selected; no max-retry protection (assumes loop exits quickly).

### updateLegsDrawingInfo
- **Signature:** `void updateLegsDrawingInfo()`
- **Purpose:** Resolve legs sprite state (action, view, frame, color) to SDL surface and rect; retry randomization on invalid data up to 100 times.
- **Inputs:** None (uses member state: `mLegsAction`, `mLegsView`, `mLegsFrame`, `mLegsColor`)
- **Outputs/Return:** None
- **Side effects:** Allocates/frees `mLegsSurface`, `mLegsData`; updates `mLegsRect`; sets `mLegsValid`, `mLegsDirty` flags.
- **Calls:** `get_player_shape_definitions()`, `get_shape_animation_data()`, `get_low_level_shape_definition()`, `get_shape_surface()`, `SDL_FreeSurface()`, `free()`, `local_random()`
- **Notes:**
  - If a state member is `NONE` (-1), a random value is chosen and `tryAgain` is set.
  - If a user-specified value is out of range, the loop breaks early (data remains invalid).
  - If random choice yields NULL pointers or fails at `get_shape_surface()`, loop continues (retry).
  - Skips shape fetch if `bit_depth == 8`; surface remains NULL.
  - `mLegsRect` origin is set from low-level shape key point (not origin point).
  - Loop exits after 100 attempts or on success; `mLegsDirty` is cleared regardless.

### updateTorsoDrawingInfo
- **Signature:** `void updateTorsoDrawingInfo()`
- **Purpose:** Resolve torso sprite state (action, pseudo-weapon, view, frame, color) to SDL surface and rect; retry randomization on invalid data up to 100 times.
- **Inputs:** None (uses member state: `mTorsoAction`, `mPseudoWeapon`, `mTorsoView`, `mTorsoFrame`, `mTorsoColor`)
- **Outputs/Return:** None
- **Side effects:** Allocates/frees `mTorsoSurface`, `mTorsoData`; updates `mTorsoRect`; sets `mTorsoValid`, `mTorsoDirty` flags.
- **Calls:** `get_player_shape_definitions()`, `get_shape_animation_data()`, `get_low_level_shape_definition()`, `get_shape_surface()`, `SDL_FreeSurface()`, `free()`, `local_random()`
- **Notes:**
  - Similar retry logic to legs; action selects torso shape array via switch statement (idle, firing, or charging).
  - `mTorsoRect` origin is set from low-level shape **origin point** (not key point, unlike legs).
  - Skips shape fetch if `bit_depth == 8`.
  - Loop exits after 100 attempts or on success; `mTorsoDirty` is cleared regardless.

### drawAt
- **Signature:** `void drawAt(SDL_Surface* inSurface, int16 inX, int16 inY)`
- **Purpose:** Composite and render legs and torso sprites onto target surface at world position.
- **Inputs:** `inSurface` (target SDL surface), `inX`, `inY` (world coordinates)
- **Outputs/Return:** None
- **Side effects:** Blits pixel data to `inSurface`; skips blit if surface is NULL.
- **Calls:** `canDrawLegs()`, `canDrawTorso()`, `SDL_BlitSurface()` (twice)
- **Notes:** Updates drawing info before blitting via `canDrawLegs/Torso()` queries; const_casts SDL_Rect for blit (mutable rect required by SDL API despite const source surface).

### objectCreated / objectDestroyed
- **Signature:** `static void objectCreated()` / `static void objectDestroyed()`
- **Purpose:** Manage collection reference counting; load/unload shape collection when first/last instance is created/destroyed.
- **Inputs:** None (modifies static counter)
- **Outputs/Return:** None
- **Side effects:** 
  - `objectCreated`: Increments counter; if 0ΓåÆ1, marks collection and loads it.
  - `objectDestroyed`: Decrements counter; if 1ΓåÆ0, unmarks collection.
- **Calls:** `get_player_shape_definitions()`, `mark_collection()`, `load_collections()`
- **Notes:** Commented-out increment in `objectCreated` suggests prior refactoring. `load_collections(false, false)` loads without progress bar or OpenGL prep.

## Control Flow Notes
Called on-demand from rendering code; no frame-loop integration. Player shape collection is marked in memory when the first `PlayerImage` is constructed and unmarked when the last is destroyed. Methods like `drawAt()` implicitly update sprite data via `canDrawLegs/Torso()` status checks, deferring expensive resolution until needed.

## External Dependencies
- **SDL:** `SDL_Surface`, `SDL_FreeSurface`, `SDL_BlitSurface`, `SDL_Rect`
- **Shape/collection system (defined elsewhere):** `get_player_shape_definitions()`, `get_shape_animation_data()`, `get_low_level_shape_definition()`, `get_shape_surface()`, `mark_collection()`, `load_collections()`
- **Random:** `local_random()`
- **Globals:** `bit_depth` (extern short), `NONE` constant, `BUILD_DESCRIPTOR`, `BUILD_COLLECTION` macros
