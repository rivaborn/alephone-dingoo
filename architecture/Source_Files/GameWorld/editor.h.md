# Source_Files/GameWorld/editor.h

## File Purpose
Header file for the map editor subsystem defining data structures, version constants, and geometric bounds. Establishes compatibility across Marathon 1/2/Infinity data formats and constrains valid map geometry.

## Core Responsibilities
- Define data version constants for Marathon series format compatibility
- Specify map coordinate bounds (min/max X/Y)
- Specify floor/ceiling height constraints with safety margins
- Define guard path control point structures (`saved_path`)
- Define map metadata structure (`map_index_data`)
- Define validation constants (max control points, lines per vertex)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `map_index_data` | struct | Metadata for a map level: name, unused padding byte, and flags |
| `saved_path` | struct | Guard patrol/AI waypoint path: point count, flags, world positions, polygon references |

## Global / File-Static State
None.

## Key Functions / Methods
None.

## Control Flow Notes
Not applicable. This file is purely data definitions and constants used by map loading/saving and the editor UI.

## External Dependencies
- **Defines used but not in this file:**
  - `WORLD_ONE` ΓÇö engine unit scale constant
  - `SHORT_MIN`, `SHORT_MAX` ΓÇö C type limits
  - `LEVEL_NAME_LENGTH` ΓÇö array size for level names
  - `world_point2d` ΓÇö 2D coordinate struct (likely defined elsewhere)

- **Version strategy:** `EDITOR_MAP_VERSION` is set to `MARATHON_INFINITY_DATA_VERSION` (value 2), indicating the editor targets Marathon Infinity format as the canonical interchange format across all three game versions.
