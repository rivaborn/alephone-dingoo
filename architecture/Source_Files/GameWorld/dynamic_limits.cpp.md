# Source_Files/GameWorld/dynamic_limits.cpp

## File Purpose
Manages configurable runtime limits for game entities (objects, monsters, projectiles, effects, paths). Allows limits to be loaded from XML configuration instead of being hardcoded, with fallback to reasonable defaults and support for resetting to original values.

## Core Responsibilities
- Maintains array of 8 dynamic limit values with defaults higher than originals
- Backs up and restores original limit values when configuration is reset
- Parses XML elements to configure individual limits with validation (0ΓÇô32767 range)
- Coordinates resizing of game entity container vectors when limits change
- Allocates pathfinding memory proportionally to configured path limits
- Provides accessor function for other modules to query current limits by type

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `XML_DynLimValueParser` | class (XML_ElementParser subclass) | Parses individual limit value attributes from XML tags like `<objects value="1024"/>` |
| `XML_DynLimParser` | class (XML_ElementParser subclass) | Root parser for `<dynamic_limits>` element; triggers container resizing on completion |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `dynamic_limits` | `uint16[NUMBER_OF_DYNAMIC_LIMITS]` (8 elements) | static | Current limit values; indexed by `_dynamic_limit_*` enum |
| `original_dynamic_limits` | `uint16*` | static | Heap allocation storing unmodified defaults; lazy-initialized on first parse |
| `DynLimParser0ΓÇô7` | `XML_DynLimValueParser` | static | Eight parser instances, each bound to one limit slot (objects, monsters, paths, projectiles, effects, rendered, local_collision, global_collision) |
| `DynamicLimitsParser` | `XML_DynLimParser` | static | Root parser instance; aggregates the 8 child parsers |

## Key Functions / Methods

### XML_DynLimValueParser::Start
- **Signature:** `bool Start()`
- **Purpose:** Initialize parser state and lazily allocate backup of original limits.
- **Inputs:** None (uses member `ValuePtr`).
- **Outputs/Return:** Always returns `true`.
- **Side effects (global state, I/O, alloc):** On first call across all instances, allocates `original_dynamic_limits` heap buffer and copies current `dynamic_limits` array into it; sets `IsPresent` flag to `false`.
- **Calls (direct calls visible in this file):** `malloc()`, `assert()`.
- **Notes:** Backup happens once per application run; subsequent calls are no-ops except for flag reset. No error recovery if malloc fails (asserts).

### XML_DynLimValueParser::HandleAttribute
- **Signature:** `bool HandleAttribute(const char *Tag, const char *Value)`
- **Purpose:** Extract and validate a single XML attribute (only "value" is recognized).
- **Inputs:** `Tag` (attribute name), `Value` (string to parse).
- **Outputs/Return:** `true` if attribute is valid and parsed; `false` if unrecognized or out of range.
- **Side effects (global state, I/O, alloc):** Updates `*ValuePtr` (dereferenced pointer to one element of `dynamic_limits[]`) and sets `IsPresent = true` on success.
- **Calls (direct calls visible in this file):** `StringsEqual()`, `ReadBoundedUInt16Value()`, `UnrecognizedTag()`.
- **Notes:** Clamps value to [0, 32767]; rejects any other attribute name.

### XML_DynLimValueParser::AttributesDone
- **Signature:** `bool AttributesDone()`
- **Purpose:** Validate that required "value" attribute was present.
- **Inputs:** None.
- **Outputs/Return:** `true` if `IsPresent` was set by `HandleAttribute()`; `false` and logs error otherwise.
- **Side effects (global state, I/O, alloc):** Calls `AttribsMissing()` on failure (logs error).
- **Calls (direct calls visible in this file):** `AttribsMissing()`.
- **Notes:** Enforces that every limit element in XML must have a `value` attribute.

### XML_DynLimParser::End
- **Signature:** `bool End()`
- **Purpose:** Resize all game entity container vectors to match newly-parsed limits.
- **Inputs:** None (reads from global `dynamic_limits[]` array).
- **Outputs/Return:** Always returns `true`.
- **Side effects (global state, I/O, alloc):** Calls `.resize()` on global vectors `EffectList`, `ObjectList`, `MonsterList`, `ProjectileList` using macros `MAXIMUM_EFFECTS_PER_MAP`, etc., which call `get_dynamic_limit()`; also calls `allocate_pathfinding_memory()`.
- **Calls (direct calls visible in this file):** `.resize()` on vectors; `allocate_pathfinding_memory()` (defined elsewhere).
- **Notes:** Commented-out code suggests historical direct calls to `objlist_clear()`. Resizing preserves existing elements if shrinking, or zero-initializes new slots if growing.

### XML_DynLimParser::ResetValues
- **Signature:** `bool ResetValues()`
- **Purpose:** Restore `dynamic_limits[]` to original defaults and trigger re-initialization.
- **Inputs:** None.
- **Outputs/Return:** Always returns `true`.
- **Side effects (global state, I/O, alloc):** If `original_dynamic_limits` is non-null, copies it back to `dynamic_limits[]`, frees the backup, nulls the pointer, and calls `End()` to resize containers.
- **Calls (direct calls visible in this file):** `End()`, `free()`.
- **Notes:** Safe to call multiple times (no-op if backup was already freed). Does not validate that backup exists; no error reporting if called before any parse.

### DynamicLimits_GetParser
- **Signature:** `XML_ElementParser *DynamicLimits_GetParser()`
- **Purpose:** Retrieve the configured root parser for XML deserialization.
- **Inputs:** None.
- **Outputs/Return:** Pointer to static `DynamicLimitsParser` instance with all 8 child parsers attached.
- **Side effects (global state, I/O, alloc):** Adds 8 child parsers to `DynamicLimitsParser.children` via repeated `AddChild()` calls; these calls are idempotent within a single invocation but may add duplicates on repeated calls (not guarded).
- **Calls (direct calls visible in this file):** `AddChild()` (8├ù).
- **Notes:** Called once at engine initialization to wire up the parser tree. Repeated calls will duplicate children (potential bug).

### get_dynamic_limit
- **Signature:** `uint16 get_dynamic_limit(int which)`
- **Purpose:** Accessor to retrieve a single limit by enum index.
- **Inputs:** `which` (enum value in range [0, NUMBER_OF_DYNAMIC_LIMITS)).
- **Outputs/Return:** Current value from `dynamic_limits[which]`.
- **Side effects (global state, I/O, alloc):** None.
- **Calls (direct calls visible in this file):** None.
- **Notes:** No bounds checking; out-of-range access is undefined. Heavily used by macros in `map.h`, `effects.h`, `monsters.h`, `projectiles.h` to define container max sizes.

## Control Flow Notes
This file participates in the **map initialization** phase:
1. At engine startup, `DynamicLimits_GetParser()` is called to register the parser with the XML system.
2. When loading a map's configuration, the XML parser invokes `Start()` on each child parser, then `HandleAttribute()` for each attribute, then `AttributesDone()`, then `End()`.
3. `End()` resizes all entity containers. This may happen at map load time or when limits are reconfigured.
4. `ResetValues()` may be called to restore defaults (e.g., when switching maps or resetting game state).
5. During gameplay, `get_dynamic_limit()` is inlined and called to determine max container sizes for new entities.

## External Dependencies
- **`cseries.h`**: Provides macros, assertions, and utility functions.
- **`dynamic_limits.h`**: Enum definitions (`_dynamic_limit_*`), `NUMBER_OF_DYNAMIC_LIMITS`, parser interface.
- **`map.h`**: Declares `MAXIMUM_OBJECTS_PER_MAP` macro using `get_dynamic_limit()`.
- **`effects.h`**: Declares `MAXIMUM_EFFECTS_PER_MAP`; vector `EffectList`.
- **`monsters.h`**: Declares `MAXIMUM_MONSTERS_PER_MAP`; vector `MonsterList`; collision buffer size macros.
- **`projectiles.h`**: Declares `MAXIMUM_PROJECTILES_PER_MAP`; vector `ProjectileList`.
- **`flood_map.h`**: Header only; declares `allocate_pathfinding_memory()` (called from `End()`).
- **`XML_ElementParser` (base class)**: Provides XML parsing framework; defined elsewhere.
- **Utility functions** (`StringsEqual`, `ReadBoundedUInt16Value`, `UnrecognizedTag`, `AttribsMissing`): Defined elsewhere in CSeries.
