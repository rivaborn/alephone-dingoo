# Source_Files/RenderOther/overhead_map.cpp

## File Purpose
Manages overhead map display configuration and XML parsing for the Aleph One game engine. Delegates rendering to platform-specific implementations (software via SDL/Quickdraw or hardware via OpenGL), and maintains global visual parameters for how the automap displays geometry, entities, and annotations.

## Core Responsibilities
- Maintain global configuration data structure (`OvhdMap_ConfigData`) with all colors, fonts, line/polygon/thing definitions, and display flags
- Parse XML configuration for overhead map appearance, monster/entity display mappings, and rendering modes
- Initialize map fonts on first use
- Select and dispatch to appropriate renderer (OpenGL or software)
- Manage automap visibility reset based on rendering mode (cumulative, currently-visible-only, or all-visible)
- Export XML parser hierarchy for integration with game's configuration system

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `OvhdMap_CfgDataStruct` | struct | Central config holding polygon/line/thing color definitions, font specs, monster display mappings, player entity size, and visibility flags |
| `XML_LiveAssignParser` | class | Parses `<assign_live>` elements mapping monster types to display classifications |
| `XML_DeadAssignParser` | class | Parses `<assign_dead>` elements mapping dead-entity collections to display classifications |
| `XML_OvhdMapBooleanParser` | class | Generic parser for boolean display-control attributes (aliens, items, projectiles, paths) |
| `XML_LineWidthParser` | class | Parses `<line>` elements to configure line widths at different zoom scales |
| `XML_OvhdMapParser` | class | Root parser for `<overhead_map>` element; manages color/font arrays and child parser integration |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `OvhdMap_ConfigData` | `OvhdMap_CfgDataStruct` | static | Central overhead map appearance config: colors (10 polygons, 3 lines, 5 things, annotations, map name, path), fonts, display flags, entity definitions |
| `MapFontsInited` | bool | static | Guard flag preventing double-initialization of overhead map fonts |
| `OGL_MapActive` | bool | global | Controls renderer selection: true uses OpenGL, false uses software renderer |
| `OverheadMap_SW` | `OverheadMap_SDL_Class` / `OverheadMap_QD_Class` | static | Software rendering engine (platform-dependent) |
| `OverheadMap_OGL` | `OverheadMap_OGL_Class` | static (conditional) | OpenGL rendering engine (when `HAVE_OPENGL` defined) |
| `OverheadMapMode` | short | static | Display mode: OverheadMap_Normal (cumulative), OverheadMap_CurrentlyVisible (reset), OverheadMap_All (all visible) |
| `LiveAssignParser` | `XML_LiveAssignParser` | static | Parser for living monster display type assignments |
| `DeadAssignParser` | `XML_DeadAssignParser` | static | Parser for dead entity collection display type assignments |
| `ShowAliensParser`, `ShowItemsParser`, `ShowProjectilesParser`, `ShowPathsParser` | `XML_OvhdMapBooleanParser` | static | Display flag parsers (4 instances) |
| `LineWidthParser` | `XML_LineWidthParser` | static | Parser for line width configuration |
| `OvhdMapParser` | `XML_OvhdMapParser` | static | Root overhead map parser |

## Key Functions / Methods

### InitMapFonts
- Signature: `static void InitMapFonts()`
- Purpose: Lazily initialize all overhead map fonts on first use
- Inputs: None
- Outputs/Return: None
- Side effects: Calls `Init()` on all annotation fonts (across all scales) and map-name font; sets `MapFontsInited = true`
- Calls: `FontSpecifier::Init()` (multiple invocations)
- Notes: Defensive initialization prevents redundant work; must be called before any text rendering

### _render_overhead_map
- Signature: `void _render_overhead_map(struct overhead_map_data *data)`
- Purpose: Main rendering entry point; selects renderer backend and triggers rendering
- Inputs: `data` ΓÇö overhead_map_data struct with mode, scale, origin point, polygon index, dimensions
- Outputs/Return: None
- Side effects: Initializes fonts if needed; selects renderer based on `OGL_MapActive` flag; sets renderer's `ConfigPtr` to global config; invokes renderer's `Render()` method
- Calls: `InitMapFonts()`, renderer's `Render()` method
- Notes: Renderer abstraction allows same config to drive both OpenGL and software rendering

### ResetOverheadMap
- Signature: `void ResetOverheadMap()`
- Purpose: Reset automap line/polygon visibility tracking according to current mode
- Inputs: None
- Outputs/Return: None
- Side effects: Uses `memset()` to reset `automap_lines` and `automap_polygons` bitsets:
  - OverheadMap_Normal: no change (cumulative visibility)
  - OverheadMap_CurrentlyVisible: clear all bits (reset each frame)
  - OverheadMap_All: set all bits to 0xff (everything visible)
- Calls: `memset()` (standard library)
- Notes: Bit count calculations: `(count/8 + ((count%8)?1:0))` bytes to cover count bits

### XML_LiveAssignParser::Start
- Signature: `bool XML_LiveAssignParser::Start()`
- Purpose: Initialize parser state when XML element begins
- Inputs: None
- Outputs/Return: true
- Side effects: Clears `IsPresent[0]` and `IsPresent[1]` flags
- Calls: None
- Notes: Trivial initialization; pattern mirrors destructor-free setup

### XML_LiveAssignParser::HandleAttribute
- Signature: `bool XML_LiveAssignParser::HandleAttribute(const char *Tag, const char *Value)`
- Purpose: Parse "monster" and "type" XML attributes
- Inputs: `Tag` ΓÇö attribute name; `Value` ΓÇö attribute value as string
- Outputs/Return: true if recognized and valid; false otherwise
- Side effects: On success, sets `Monster` (int16 bounded [0, NUMBER_OF_MONSTER_TYPES-1]) and `Type` (int16 bounded [-1, 1]); sets corresponding `IsPresent` flags
- Calls: `StringsEqual()`, `ReadBoundedInt16Value()`, `UnrecognizedTag()`
- Notes: Type values map to NONE/-1, _civilian_thing/0, _monster_thing/1

### XML_LiveAssignParser::AttributesDone
- Signature: `bool XML_LiveAssignParser::AttributesDone()`
- Purpose: Validate required attributes and commit parsed assignment to config
- Inputs: None
- Outputs/Return: true if both attributes present; false otherwise
- Side effects: Updates `OvhdMap_ConfigData.monster_displays[Monster] = Type` if validation passes
- Calls: `AttribsMissing()` on validation failure
- Notes: Requires both "monster" and "type" attributes; called by XML parser after all attributes for element parsed

### XML_OvhdMapParser::Start
- Signature: `bool XML_OvhdMapParser::Start()`
- Purpose: Prepare parser state and set up color/font arrays for XML modification
- Inputs: None
- Outputs/Return: true
- Side effects: Resets `GotUsed` flags on 4 child boolean parsers; copies current `OvhdMap_ConfigData` colors and fonts into local `Colors[]` and `Fonts[]` arrays via pointer iteration; calls `Color_SetArray()` and `Font_SetArray()` to register arrays for color/font parsers
- Calls: `Color_SetArray()`, `Font_SetArray()`
- Notes: Complex pointer arithmetic iterates polygon colors, line colors, thing colors, annotation colors, map-title color, and path color; mirrors array construction in `End()`

### XML_OvhdMapParser::HandleAttribute
- Signature: `bool XML_OvhdMapParser::HandleAttribute(const char *Tag, const char *Value)`
- Purpose: Parse "mode" and "title_offset" attributes for overhead_map element
- Inputs: `Tag` ΓÇö attribute name; `Value` ΓÇö attribute value string
- Outputs/Return: true if valid; false otherwise
- Side effects: Sets `OverheadMapMode` (bounded [0, NUMBER_OF_OVERHEAD_MAP_MODES-1]) or `OvhdMap_ConfigData.map_name_data.offset_down` (int16)
- Calls: `StringsEqual()`, `ReadBoundedInt16Value()`, `ReadInt16Value()`, `UnrecognizedTag()`
- Notes: Mode attribute controls reset behavior; offset_down is map title vertical positioning

### XML_OvhdMapParser::End
- Signature: `bool XML_OvhdMapParser::End()`
- Purpose: Finalize parsing and write modified colors/fonts back to config
- Inputs: None
- Outputs/Return: true
- Side effects: Checks child parser `GotUsed` flags and updates config display flags (ShowAliens, ShowItems, ShowProjectiles, ShowPaths); copies modified colors and fonts from local arrays back to `OvhdMap_ConfigData` via pointer iteration
- Calls: None
- Notes: Mirrors `Start()` logic in reverse; preserves arrays' pointer-based iteration structure

### OverheadMap_GetParser
- Signature: `XML_ElementParser *OverheadMap_GetParser()`
- Purpose: Build and return root overhead map XML parser hierarchy
- Inputs: None
- Outputs/Return: Pointer to `OvhdMapParser` (root parser instance)
- Side effects: Adds 8 child parsers to `OvhdMapParser` (LiveAssignParser, DeadAssignParser, 4 boolean display-flag parsers, LineWidthParser, Color_GetParser(), Font_GetParser())
- Calls: `OvhdMapParser.AddChild()` (8 times), `Color_GetParser()`, `Font_GetParser()`
- Notes: Single entry point for all overhead map configuration parsing

## Control Flow Notes
- **Initialization**: `InitMapFonts()` is called defensively by `_render_overhead_map()` before rendering
- **Rendering dispatch**: `_render_overhead_map()` ΓåÆ renderer selection based on `OGL_MapActive` flag ΓåÆ renderer's `Render()` method
- **Configuration flow**: XML parser hierarchy populates `OvhdMap_ConfigData` at load time; renderers read this static config
- **Visibility tracking**: `ResetOverheadMap()` called externally (e.g., level load); bitset updates control which map elements are visible
- **Mode semantics**: OverheadMapMode affects automap reset behavior and thus long-term visibility accumulation

## External Dependencies
- **Includes**: `cseries.h` (cross-platform utilities), `shell.h` (_get_player_color), `map.h` (world geometry, automap bitsets), `monsters.h` (NUMBER_OF_MONSTER_TYPES), `overhead_map.h` (public interface), `player.h` (player info), `render.h` (rendering integration), `flood_map.h` (pathfinding types), `platforms.h` (platform data), `media.h` (media types), `ColorParser.h` (RGB color parsing)
- **Platform-conditional includes**: `OverheadMap_QD.h` (Mac Quickdraw renderer), `OverheadMap_SDL.h` (SDL software renderer), `OverheadMap_OGL.h` (OpenGL renderer)
- **External symbols**: `dynamic_world` (world state for line/polygon counts), `automap_lines`, `automap_polygons` (visibility bitsets), `Color_GetParser()`, `Font_GetParser()` (color and font XML parsers), renderer classes (`OverheadMapClass`, `OverheadMap_SDL_Class`, etc.)
