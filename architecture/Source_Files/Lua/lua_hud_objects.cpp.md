# Source_Files/Lua/lua_hud_objects.cpp

## File Purpose
Implements Lua bindings for HUD (heads-up display) objects and engine globals, exposing graphical rendering (images, shapes, fonts), screen properties (dimensions, clip regions, field of view), and game state (players, difficulty, scoring) to Lua scripts. Supports creation, manipulation, and drawing of HUD primitives.

## Core Responsibilities
- Register Lua enum types and container classes for game concepts (item types, player colors, game types, difficulty levels, renderer types, texture types, fade effects)
- Provide Lua wrappers for blitter classes (`Image_Blitter`, `Shape_Blitter`) with property getters/setters (dimensions, tint, rotation, crop rectangles)
- Expose screen/view properties to Lua (window dimensions, clip rect, world rect, map rect, terminal rect, field of view, renderer type, UI scale preferences)
- Implement player and game query APIs (player list, kills, ranking, color, team, difficulty, time remaining, scoring mode, game ticks, version)
- Handle HUD drawing commands (fill/frame rectangles, draw images/shapes/text) via `Lua_HUDInstance()`
- Manage color lookups supporting both named (r/g/b/a) and indexed (1/2/3/4) color channel access
- Register lighting fader queries (liquid/damage effects with type, color, active state)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `Lua_Image` | typedef (L_ObjectClass wrapper) | Lua interface to `Image_Blitter*` with properties |
| `Lua_Shape` | typedef (L_ObjectClass wrapper) | Lua interface to `Shape_Blitter*` with properties |
| `Lua_Font` | typedef (L_ObjectClass wrapper) | Lua interface to `FontSpecifier*` for text measurement/drawing |
| `Lua_Image_Crop_Rect` / `Lua_Shape_Crop_Rect` | class | Provides access to crop rectangle (x, y, width, height) of image/shape blitters |
| `Lua_Screen_Clip_Rect`, `Lua_Screen_World_Rect`, `Lua_Screen_Map_Rect`, `Lua_Screen_Term_Rect` | typedef | Lua proxies for screen region properties |
| `Lua_Screen_FOV` | typedef | Field of view properties (horizontal, vertical, fix mode) |
| `Lua_HUDGame_Player` | typedef | Lua proxy for single player data (color, team, name, kills, ranking) |
| `Lua_HUDGame_Players` | typedef | Indexed container for all players with `__len` and `__index` metamethods |
| `Lua_HUDGame` | typedef | Game state proxy (difficulty, kill limit, time remaining, ticks, type, scoring mode, version) |
| `Lua_HUDLighting_Fader` | typedef | Lighting effect state (active, type, color) |
| `Lua_HUDLighting` | typedef | Ambient and weapon lighting queries |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `world_view` | `struct view_data*` | extern | Current camera/viewport state (used by FOV and map/term rect getters) |
| `local_player_index` | short | extern | Index of the local (human-controlled) player |
| `current_player` | `struct player_data*` | extern | Pointer to current player being queried |
| `dynamic_world` | struct | extern | Global game world state (player count, game info, tick count) |
| `game_scoring_mode` | int | extern | Current game scoring mode enumeration |
| `AngleConvert` | const float | static | Conversion constant (360 / FULL_CIRCLE) for angle normalization |
| Lua type name strings (e.g., `Lua_Image_Name`, `Lua_Shape_Name`) | char[] | static | Metatable registration identifiers for Lua class system |

## Key Functions / Methods

### Lua_HUDColor_Lookup
- **Signature:** `static float Lua_HUDColor_Lookup(lua_State *L, int pos, int idx, const char *name1, const char *name2)`
- **Purpose:** Extract a single color channel (RGBA) from a Lua table by trying named keys first, then numeric index
- **Inputs:** Lua state, stack position of color table, fallback numeric index, two alternative field names (e.g., "r"/"red")
- **Outputs/Return:** Float value [0.0, 1.0] for the channel; defaults to 1.0 if not found
- **Side effects:** Pops temporary values from Lua stack
- **Calls:** `lua_pushstring`, `lua_gettable`, `lua_isnil`, `lua_pop`, `lua_tonumber`
- **Notes:** Supports both short and long color channel names; flexible table schema for Lua scripts

### Lua_HUDColor_Get_R/G/B/A
- **Signature:** `static float Lua_HUDColor_Get_{R|G|B|A}(lua_State *L, int pos)`
- **Purpose:** Convenience wrappers extracting individual color channels
- **Inputs:** Lua state, stack position of color table
- **Outputs/Return:** Float [0.0, 1.0] for red/green/blue/alpha
- **Calls:** `Lua_HUDColor_Lookup`

### Lua_Images_New
- **Signature:** `int Lua_Images_New(lua_State *L)`
- **Purpose:** Factory function to create a new `Image_Blitter` from Lua, supporting both file paths and engine resources
- **Inputs:** Lua table with optional fields: `resource` (int ID) or `path` (string file path), `mask` (optional mask file path)
- **Outputs/Return:** Pushes new `Lua_Image` instance or nil on failure
- **Side effects:** Allocates new `Image_Blitter` or `OGL_Blitter` (if OpenGL enabled); loads image from file or resource
- **Calls:** `Lua_Image::Push`, `get_screen_mode`, `new Image_Blitter`, `new OGL_Blitter`, `ImageDescriptor::LoadFromFile`
- **Notes:** Handles OpenGL vs. software rendering selection; optional alpha mask loading

### Lua_Shapes_New
- **Signature:** `int Lua_Shapes_New(lua_State *L)`
- **Purpose:** Factory for `Shape_Blitter` from collection textures
- **Inputs:** Lua table with fields: `collection` (Lua_Collection), `texture_index` (short), `type` (Lua_TextureType), `color_table` (short CLUT index)
- **Outputs/Return:** Pushes new `Lua_Shape` instance or nil if width is zero
- **Side effects:** Allocates `Shape_Blitter`
- **Calls:** `Lua_Collection::ToIndex`, `Lua_TextureType::ToIndex`, `new Shape_Blitter`
- **Notes:** Validates texture and color-table indices against engine maximums; returns nil for invalid geometry

### Lua_Image_Draw / Lua_Shape_Draw
- **Signature:** `int Lua_Image_Draw(lua_State *L)` / `int Lua_Shape_Draw(lua_State *L)`
- **Purpose:** Queue image/shape for rendering at specified screen coordinates
- **Inputs:** Lua stack: self (image/shape), x (float), y (float)
- **Outputs/Return:** 0 (no return value)
- **Side effects:** Calls `Lua_HUDInstance()->draw_image()` or `draw_shape()` to enqueue draw command
- **Calls:** `Lua_HUDInstance`, `draw_image`, `draw_shape`

### Lua_Screen_Fill_Rect / Lua_Screen_Frame_Rect
- **Signature:** `int Lua_Screen_Fill_Rect(lua_State *L)` / `int Lua_Screen_Frame_Rect(lua_State *L)`
- **Purpose:** Draw filled or outlined rectangles with specified RGBA color
- **Inputs:** x, y, width, height (floats), color table (RGBA), optional thickness (frame only)
- **Outputs/Return:** 0
- **Side effects:** Calls HUD renderer for immediate/queued drawing
- **Calls:** `Lua_HUDColor_Get_{R|G|B|A}`, `Lua_HUDInstance`

### Lua_HUDGame_Get_Players / Lua_HUDGame_Players_Get / Lua_HUDGame_Players_Length
- **Signature:** Player container access functions
- **Purpose:** Expose indexed player list with length and __index metamethods
- **Inputs:** Lua state, player index (integer)
- **Outputs/Return:** Pushes `Lua_HUDGame_Player` instance or provides player count
- **Calls:** `dynamic_world->player_count`, `Lua_HUDGame_Player::Push`
- **Notes:** Validates index range; enables Lua iteration and indexing like `Game.players[0]` or `#Game.players`

### Lua_HUDObjects_register
- **Signature:** `int Lua_HUDObjects_register(lua_State *L)`
- **Purpose:** Master registration function; sets up all Lua enum types, object classes, and global instances
- **Inputs:** Lua state
- **Outputs/Return:** 0 (success)
- **Side effects:** Registers ~20 enum classes, ~10 object classes, 5+ global instances (Player, Screen, Game, Fonts, Images, Shapes, Lighting); populates mnemonic tables for mnemonics (if provided)
- **Calls:** Many `::Register()` calls on Lua class types; `::Push()` to instantiate globals; `lua_setglobal` to bind globals to Lua environment
- **Notes:** This is the entry point for bootstrapping the entire HUD Lua API; called once during engine initialization

## Control Flow Notes
This file is purely declarative and is called during **initialization** (not frame or render loops). The `Lua_HUDObjects_register()` function is invoked once to expose all HUD-related symbols to the Lua interpreter. Drawing commands (`Lua_Image_Draw`, `Lua_Screen_Fill_Rect`, etc.) are queued via `Lua_HUDInstance()` and are typically executed later in the render pipeline. Property getters dynamically query engine state (`world_view`, `dynamic_world`) to reflect real-time game data when accessed from Lua scripts.

## External Dependencies
- **Lua C API:** `lua.h`, `lauxlib.h`, `lualib.h` (core interpreter and auxiliary library functions)
- **Game engine:** `items.h`, `player.h`, `motion_sensor.h`, `screen.h`, `shell.h`, `network.h`, `render.h` (game world data)
- **Rendering:** `HUDRenderer.h`, `HUDRenderer_Lua.h`, `Image_Blitter.h`, `OGL_Blitter.h`, `Shape_Blitter.h`, `FontHandler.h` (blitters and HUD rendering)
- **Effects/lighting:** `fades.h`, `OGL_Faders.h` (visual effects system)
- **Resources:** `collection_definition.h`, `alephversion.h` (asset management)
- **Templates:** `lua_templates.h`, `lua_objects.h`, `lua_map.h` (Lua binding framework and other domain bindings)
- **Boost:** `boost/bind.hpp`, `boost/shared_ptr.hpp` (utilities, though `shared_ptr` not used in this file)
