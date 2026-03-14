# Source_Files/GameWorld/dynamic_limits.h

## File Purpose
Defines the interface for querying and configuring dynamic runtime limits on game entities (objects, monsters, projectiles, effects, etc.). Supports XML-based configuration loading instead of resource forks. Provides centralized access to entity count constraints that affect AI pathfinding, rendering, and collision systems.

## Core Responsibilities
- Enumerate all dynamic limit types (objects, NPCs, projectiles, effects, collision buffers, rendering queue)
- Declare XML parser factory for loading limits from configuration files
- Provide runtime accessor for querying specific entity count limits
- Support both local and global collision buffer capacity constraints

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `_dynamic_limit_*` (enum values) | enum | Identifies constraint categories: objects, monsters, paths, projectiles, effects, rendered entities, and collision buffers |

## Global / File-Static State
None.

## Key Functions / Methods

### DynamicLimits_GetParser
- Signature: `XML_ElementParser *DynamicLimits_GetParser()`
- Purpose: Factory function to obtain the parser object for XML-based dynamic limit configuration
- Inputs: None
- Outputs/Return: Pointer to XML_ElementParser instance for dynamic limits
- Side effects: Likely allocates parser object (deallocation responsibility unclear from header)
- Calls: Not inferable from this file
- Notes: Implementation must be in a .cpp file; enables XML file initialization per 2000-05-04 change note

### get_dynamic_limit
- Signature: `uint16 get_dynamic_limit(int which)`
- Purpose: Query the current runtime limit for a specified constraint category
- Inputs: `which` ΓÇö index into limit type enum (e.g., `_dynamic_limit_objects`)
- Outputs/Return: uint16 representing the entity count cap for that category
- Side effects: None (pure accessor)
- Calls: Not inferable from this file
- Notes: No bounds checking visible; caller must use valid enum values

## Control Flow Notes
This is an **initialization/configuration-time interface**. The XML parser is invoked during engine startup to load limit values from config files. The `get_dynamic_limit` accessor is called at runtime (frame loop, object spawning, AI pathfinding) to enforce entity quotas. The collision buffer limits (local [16], global [64]) suggest hard-coded tuning for spatial query performance.

## External Dependencies
- **XML_ElementParser.h** ΓÇö Base class for XML parsing infrastructure
- **cstypes.h** ΓÇö Type definitions (uint16, int)
- **"COPYING"** ΓÇö GPL license file reference
