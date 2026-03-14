# Source_Files/RenderOther/overhead_map.h

## File Purpose
Header file defining the overhead map rendering system for the Aleph One game engine. Declares the data structure, rendering function, and XML configuration parser for in-game overhead/mini-map display across multiple contexts (gameplay, saved game preview, checkpoint view).

## Core Responsibilities
- Define constants for overhead map scale constraints
- Declare the `overhead_map_data` struct to hold map rendering state and parameters
- Provide the main overhead map rendering entry point
- Supply XML parsing interface for overhead map configuration elements
- Support multiple rendering modes (saved game, checkpoint, live game)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `overhead_map_data` | struct | Encapsulates all parameters for a single overhead map render pass: display mode, scale factor, origin point, viewport dimensions, polygon context, and rendering flags |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `OVERHEAD_MAP_MINIMUM_SCALE` | #define (int 1) | static | Lower bound on map zoom scale to prevent excessive detail |
| `OVERHEAD_MAP_MAXIMUM_SCALE` | #define (int 4) | static | Upper bound on map zoom scale |
| `DEFAULT_OVERHEAD_MAP_SCALE` | #define (int 3) | static | Default zoom level when map is initialized |

## Key Functions / Methods

### _render_overhead_map
- Signature: `void _render_overhead_map(struct overhead_map_data *data)`
- Purpose: Renders the overhead map to the display using the provided configuration and parameters
- Inputs: Pointer to `overhead_map_data` struct containing viewport, scale, origin, and mode
- Outputs/Return: void
- Side effects: Display/rendering I/O; reads world/polygon data based on origin and viewport
- Calls: Not visible in this file (implementation in corresponding .cpp file)
- Notes: Takes ownership of parameter struct; mode field determines rendering context (preview vs. live map)

### OverheadMap_GetParser
- Signature: `XML_ElementParser *OverheadMap_GetParser()`
- Purpose: Factory function to retrieve the XML element parser for "overhead_map" configuration elements
- Inputs: None
- Outputs/Return: Pointer to an `XML_ElementParser` subclass (defined elsewhere)
- Side effects: None (accessor)
- Calls: Not visible in this file
- Notes: Added in May 2000 per Loren Petrich; supports XML-based configuration of overhead map behavior

## Control Flow Notes
This header is part of the **render subsystem**. The three modes suggest the overhead map is rendered in different contexts: during active gameplay (`_rendering_game_map`), when previewing saved games (`_rendering_saved_game_preview`), and during checkpoint/intermission sequences (`_rendering_checkpoint_map`). The file provides the interface; the implementation likely reads world polygon and entity data to populate the map view.

## External Dependencies
- **`XML_ElementParser.h`** ΓÇô Base class for XML element parsing; supports dynamic configuration of overhead map parameters
- **Implicit dependencies** (defined elsewhere):
  - `world_point2d` ΓÇô 2D world coordinate type
  - World/polygon data structures for map rendering

## Notes
- Copyright extends from 1991ΓÇô2001 by Bungie Studios and Aleph One developers
- XML parser support added May 2000 (Loren Petrich)
- `origin_polygon_index` suggests the map origin is anchored to a specific polygon in the world geometry
- The `draw_everything` flag likely controls whether only visible/exploredregions are shown vs. the entire map
