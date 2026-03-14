# Source_Files/RenderOther/ViewControl.cpp

## File Purpose
Manages camera/view control settings for the game engine, including field-of-view parameters, teleportation visual effects, landscape texture options, and on-screen rendering configuration. Provides XML-based configuration system for customizing view behavior.

## Core Responsibilities
- Maintain FOV settings (normal, extra vision, tunnel vision) and smooth FOV transitions
- Control optional visual/audio effects for teleportation (folding, static, interlevel effects)
- Store and retrieve landscape texture options indexed by collection/frame pairs
- Parse and apply view-related settings from XML configuration
- Manage on-screen display font initialization and access

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `view_settings_definition` | struct | Holds boolean flags controlling map visibility, fold/static effects, and interlevel teleport effects |
| `FOV_settings_definition` | struct | Holds FOV angles (normal, extra, tunnel), change rate, and horizontal/vertical fix setting |
| `LandscapeOptions` | struct | Defines landscape texture rendering: scale exponents, repeat mode, aspect ratio, azimuth rotation |
| `LandscapeOptionsEntry` | struct | Pairs a frame index with LandscapeOptions for per-frame texture configuration |
| `XML_FOVParser` | class | XML element parser for FOV settings attributes |
| `XML_ViewParser` | class | XML element parser for view settings (map, effects) |
| `XML_LandscapeParser` | class | XML element parser for landscape texture options |
| `XML_LO_ClearParser` | class | XML element parser to clear landscape option collections |
| `FontSpecifier` | struct (external) | Encapsulates font name, size, style, and other rendering parameters |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `view_settings` | `view_settings_definition` | global | Current view control settings; defaults enable map, fold, static, and all teleport effects |
| `FOV_settings` | `FOV_settings_definition` | global | Current FOV parameters; defaults: normal 80┬░, extra 130┬░, tunnel 30┬░, change rate 50┬░/s |
| `OnScreenFont` | `FontSpecifier` | static | On-screen display font (Monaco, 12pt); lazily initialized |
| `ScreenFontInited` | bool | static | Tracks whether `OnScreenFont` has been initialized |
| `LOList[]` | `vector<LandscapeOptionsEntry>[]` | static | Separate landscape option vectors for each of NUMBER_OF_COLLECTIONS; indexed by collection ID |
| `DefaultLandscape` | `LandscapeOptions` | static | Default landscape options returned when no match found (2├ù scales, no repeat, azimuth 0) |
| `original_FOV_settings` | pointer | static | Backup of FOV settings for XML parsing reset; allocated on first parse |
| `original_view_settings` | pointer | static | Backup of view settings for XML parsing reset; allocated on first parse |
| `FOVParser` | `XML_FOVParser` | static | Parser instance for FOV XML elements |
| `ViewParser` | `XML_ViewParser` | static | Parser instance for view XML elements |
| `LO_ClearParser` | `XML_LO_ClearParser` | static | Parser instance for landscape-clear XML elements |
| `LandscapeParser` | `XML_LandscapeParser` | static | Parser instance for landscape XML elements |
| `LandscapesParser` | `XML_ElementParser` | static | Root parser instance for landscapes XML container |

## Key Functions / Methods

### View_AdjustFOV
- **Signature:** `bool View_AdjustFOV(float& FOV, float FOV_Target)`
- **Purpose:** Smoothly transition current FOV toward a target value at a fixed rate per frame.
- **Inputs:** Reference to current FOV value; target FOV to approach
- **Outputs/Return:** `true` if FOV changed; `false` if already at target
- **Side effects:** Modifies FOV parameter in-place; ensures `FOV_ChangeRate` is positive
- **Calls:** `MAX()`, `MIN()` macros
- **Notes:** Clamps FOV to target when close to avoid overshoot; should be called once per frame during transitions

### View_GetLandscapeOptions
- **Signature:** `LandscapeOptions *View_GetLandscapeOptions(shape_descriptor Desc)`
- **Purpose:** Retrieve landscape texture options for a given shape descriptor (collection + frame).
- **Inputs:** Shape descriptor encoding collection and frame IDs
- **Outputs/Return:** Pointer to matching `LandscapeOptions`; returns `&DefaultLandscape` if not found
- **Side effects:** None
- **Calls:** `GET_DESCRIPTOR_SHAPE()`, `GET_DESCRIPTOR_COLLECTION()`, `GET_COLLECTION()` macros; vector iterator traversal
- **Notes:** Matches on exact frame or wildcard `AnyFrame` (-1); linear search within collection vector

### GetOnScreenFont
- **Signature:** `FontSpecifier& GetOnScreenFont()`
- **Purpose:** Return the on-screen display font, initializing it on first call.
- **Inputs:** None
- **Outputs/Return:** Reference to static `OnScreenFont`
- **Side effects:** Calls `OnScreenFont.Init()` and sets `ScreenFontInited` flag on first invocation
- **Calls:** `FontSpecifier::Init()` (lazy init pattern)
- **Notes:** Subsequent calls skip initialization; safe for repeated access

### XML_FOVParser::Start
- **Signature:** `bool XML_FOVParser::Start()`
- **Purpose:** Initialize FOV parser state; back up current FOV settings before parsing new values.
- **Inputs:** None
- **Outputs/Return:** `true` on success
- **Side effects:** Allocates `original_FOV_settings` if not already allocated; copies current `FOV_settings` to backup
- **Calls:** `malloc()`, `assert()` (on allocation failure)
- **Notes:** Enables reset-to-defaults if XML parse fails

### XML_FOVParser::HandleAttribute
- **Signature:** `bool XML_FOVParser::HandleAttribute(const char *Tag, const char *Value)`
- **Purpose:** Parse and validate FOV XML attributes (`normal`, `extra`, `tunnel`, `rate`, `fix_h_not_v`).
- **Inputs:** XML attribute tag name; string value
- **Outputs/Return:** `true` if attribute parsed successfully; `false` otherwise
- **Side effects:** Updates global `FOV_settings` struct members if valid
- **Calls:** `StringsEqual()`, `ReadBoundedNumericalValue()`, `ReadBooleanValueAsBool()`, `UnrecognizedTag()`
- **Notes:** Bounds check enforces 0ΓÇô180┬░ for angles and rates

### XML_ViewParser::Start
- **Signature:** `bool XML_ViewParser::Start()`
- **Purpose:** Initialize view parser; back up settings and set up font array before parsing.
- **Inputs:** None
- **Outputs/Return:** `true` on success
- **Side effects:** Allocates `original_view_settings` backup; calls `Font_SetArray(&OnScreenFont)`
- **Calls:** `malloc()`, `assert()`, `Font_SetArray()`

### XML_LandscapeParser::AttributesDone
- **Signature:** `bool XML_LandscapeParser::AttributesDone()`
- **Purpose:** Finalize landscape option entry after attributes parsed; add or replace in collection list.
- **Inputs:** None (uses member variables `Collection`, `Frame`, `Data`, `IsPresent`)
- **Outputs/Return:** `true` on success; `false` if required collection attribute missing
- **Side effects:** Searches `LOList[Collection]` for matching frame; replaces existing or appends new `LandscapeOptionsEntry`
- **Calls:** Vector `push_back()`, iterator traversal
- **Notes:** Validates `IsPresent` flag; exact frame match preferred, but new entries always added if not found

### Landscapes_GetParser / View_GetParser
- **Signature:** `XML_ElementParser *Landscapes_GetParser()` / `XML_ElementParser *View_GetParser()`
- **Purpose:** Return root XML parser objects for landscape and view configuration sections.
- **Inputs:** None
- **Outputs/Return:** Pointer to static parser instance
- **Side effects:** `View_GetParser()` adds child parsers (FOVParser, Font parser) to ViewParser
- **Calls:** `XML_ElementParser::AddChild()`
- **Notes:** Entry points for XML configuration system; called during initialization

## Control Flow Notes
This file is primarily a **configuration and state-management module**, not part of the frame/update/render loop:
- **Initialization:** XML parsers are invoked during game startup to load view configuration from XML files
- **Per-frame:** `View_AdjustFOV()` called each frame to smoothly transition FOV values (e.g., entering/exiting tunnel vision)
- **On-demand:** `View_GetLandscapeOptions()` called during landscape background rendering to fetch texture parameters
- **Lazy init:** `GetOnScreenFont()` called first time on-screen text is needed; subsequent calls are no-op
- **No direct render:** File does not render; only supplies configuration data to rendering/display code

## External Dependencies
- **Includes:** `<vector>`, `<string.h>` (STL); `"cseries.h"` (macros, types); `"world.h"` (angle, shape descriptor macros); `"shell.h"`, `"screen.h"`, `"ViewControl.h"`, `"SoundManager.h"` (interfaces)
- **Used elsewhere:** `StringsEqual()`, `ReadBoundedNumericalValue()`, `ReadBooleanValueAsBool()`, `ReadBoundedInt16Value()`, `ReadFloatValue()`, `ReadInt16Value()` (XML parsing utilities); `get_screen_mode()` (screen mode query); `Font_SetArray()`, `Font_GetParser()` (font system); shape descriptor macros (`GET_DESCRIPTOR_SHAPE`, `GET_COLLECTION`, `MAXIMUM_SHAPES_PER_COLLECTION`); angle constants (`FULL_CIRCLE`)
- **Defined elsewhere:** `XML_ElementParser` base class, `FontSpecifier`, `LandscapeOptions` (declared in ViewControl.h header)
