# Source_Files/Misc/PlayerImage_sdl.h

## File Purpose
Defines the `PlayerImage` class for managing and rendering player character sprites in SDL-based rendering. Handles sprite state (view angle, colors, team affiliation, animation frames, brightness), lazy-loads sprite graphics from resource collections, and draws player icons at specified coordinates.

## Core Responsibilities
- Manage player sprite appearance state (legs/torso view, colors, actions, animation frames, brightness, scale)
- Track and flag dirty state for efficient redraw-only-when-changed patterns
- Lazy-load SDL surfaces and sprite data from game resource collections on demand
- Maintain validity tracking for rendered leg and torso graphics
- Provide setter methods with automatic state-change detection
- Draw complete or partial player images to SDL surfaces at specified positions
- Manage collection resource marking/unmarking across object lifetime

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `PlayerImage` | class | Encapsulates player sprite state, drawing data, and rendering logic |
| `SDL_Surface` | struct (external) | Source image buffer for legs and torso sprites |
| `SDL_Rect` | struct (external) | Rectangular regions defining draw positions and source clipping |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `sNumOutstandingObjects` | `int16` | class static | Count of active `PlayerImage` instances; used to track when to mark/unmark sprite collections in resource manager |

## Key Functions / Methods

### Constructor (default)
- **Signature:** `PlayerImage()`
- **Purpose:** Initialize a player image with randomly-chosen view, color, action, and frame values
- **Inputs:** None
- **Outputs/Return:** Constructed object with all state members initialized and dirty flags set
- **Side effects:** Calls `objectCreated()` to increment resource tracking counter
- **Notes:** Expensive sprite fetching is deferred until `updateLegsDrawingInfo()` or `updateTorsoDrawingInfo()` are called; random values are finalized only at update time

### updateLegsDrawingInfo / updateTorsoDrawingInfo
- **Signature:** `void updateLegsDrawingInfo()` / `void updateTorsoDrawingInfo()`
- **Purpose:** Synchronize cached drawing data (SDL surfaces, rects, pixel data) with current state; load sprites from collection if needed
- **Inputs:** None (reads current state members)
- **Outputs/Return:** Updates `mLegsSurface`/`mTorsoSurface`, `mLegsData`/`mTorsoData`, `mLegsValid`/`mTorsoValid`
- **Side effects:** Resource collection access; potential allocation via `malloc()`/`free()`; marks/unmarks collections
- **Calls:** Not inferable from this file
- **Notes:** Called only when dirty flag is set; if resource load fails or values are out-of-range, validity flag remains false but object continues to exist

### drawAt
- **Signature:** `void drawAt(SDL_Surface* inSurface, int16 inX, int16 inY)`
- **Purpose:** Render the player image (legs and/or torso) to a destination surface at specified coordinates
- **Inputs:** Target SDL surface, X and Y destination coordinates
- **Outputs/Return:** None (modifies target surface pixels)
- **Side effects:** Calls `updateDrawingInfo()` to ensure drawing data is current; may trigger expensive sprite loading
- **Calls:** `updateDrawingInfo()`, then SDL blitting/copying operations (not visible in this file)
- **Notes:** Draws only valid components (e.g., if only legs are valid, torso is skipped)

### Setter methods (setLegsView, setTorsoView, setLegsColor, etc.)
- **Purpose:** Update individual state members with automatic dirty-flag management
- **Pattern:** All setters check if new value differs from current; if so, update state and set corresponding dirty flag(s)
- **Notes:** Value of `NONE` (-1) triggers random selection at update time; setters do not validate ranges (validation deferred to update phase)

### Status query methods (canDrawLegs, canDrawTorso, canDrawPlayer, etc.)
- **Purpose:** Check whether rendering data is valid before drawing
- **Pattern:** All call `updateDrawingInfo()` first to ensure state is current, then return validity flags
- **Notes:** Automatic update ensures caller always gets current information

## Control Flow Notes
Follows a **lazy-loading, dirty-flag** pattern:
1. **State phase:** Setters modify state members and mark dirty flags without triggering expensive operations.
2. **Update phase:** Explicit calls to `updateLegsDrawingInfo()`, `updateTorsoDrawingInfo()`, or `updateDrawingInfo()` (or triggered implicitly by query/draw methods) load sprites from resource collections and cache surfaces.
3. **Render phase:** `drawAt()` uses cached surfaces to composite legs/torso onto a destination surface.
4. **Lifecycle:** `objectCreated()` and `objectDestroyed()` track instance count for resource collection reference counting.

Not inferable: entry point into frame/update/render loop; assumed to be called by higher-level rendering or UI system.

## External Dependencies
- **Includes:** `cseries.h` (defines SDL types, int16, byte, etc.)
- **Referenced but not included:** `player.h` (leg action constants like `_player_running`), `weapons.h` (torso action and weapon constants)
- **SDL symbols:** `SDL_Surface`, `SDL_Rect`, SDL blitting functions (used in `drawAt`, not visible in this file)
