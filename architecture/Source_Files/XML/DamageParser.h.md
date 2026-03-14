# Source_Files/XML/DamageParser.h

## File Purpose
Declares the XML parser interface for damage element definitions. Parses `<damage>` XML elements and populates `damage_definition` structures with damage type, behavior flags, and scaling parameters. Part of the Aleph One engine's data-driven configuration system.

## Core Responsibilities
- Provide factory function for damage element parser creation
- Set the target damage structure to be populated by parsed XML
- Enable XML parsing of damage attributes (type, flags, base, random, scale)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `damage_definition` | struct | Defines damage parameters: type enum, alien flag, base/random amounts, fixed-point scale factor |

*(Defined in map.h; included via bundled header)*

## Global / File-Static State
None.

## Key Functions / Methods

### Damage_GetParser
- **Signature:** `XML_ElementParser *Damage_GetParser()`
- **Purpose:** Factory function returning a configured parser for `<damage>` XML elements.
- **Inputs:** None.
- **Outputs/Return:** Pointer to an XML_ElementParser instance configured for damage elements.
- **Side effects:** Likely allocates/returns a shared or cached parser instance.
- **Calls:** Not visible in this file; implementation in corresponding `.cpp`.
- **Notes:** Comments indicate the parser reads attributes named `type`, `flags`, `base`, `random`, `scale` (all optional).

### Damage_SetPointer
- **Signature:** `void Damage_SetPointer(damage_definition *DamagePtr)`
- **Purpose:** Binds the parser to a target damage structure, so subsequent parse operations populate this structure's fields.
- **Inputs:** `DamagePtr` ΓÇô pointer to the damage_definition to fill.
- **Outputs/Return:** None (void).
- **Side effects:** Stores pointer in parser state for later attribute/element processing.
- **Calls:** Not visible in this file.
- **Notes:** Must be called before parsing a damage element; allows parser reuse across multiple damage instances.

## Control Flow Notes
Likely part of a higher-level XML configuration loading pipeline. Expected usage:
1. Call `Damage_GetParser()` once per session or per level load.
2. For each `<damage>` element encountered:
   - Call `Damage_SetPointer(&target_damage)` to bind the structure.
   - Feed element to parser (via XML_ElementParser's `Start()`, `HandleAttribute()`, `End()`).
3. Parsed structure is then used by game logic (e.g., `calculate_damage()` in map.h).

## External Dependencies
- **map.h**: Defines `damage_definition` struct, damage type enum (`_damage_explosion`, etc.), and damage flag enum (`_alien_damage`).
- **XML_ElementParser.h**: Base parser class providing attribute/element handling framework.
- Aleph One game engine internals (implementation in corresponding `.cpp`).
