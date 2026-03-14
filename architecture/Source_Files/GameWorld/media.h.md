# Source_Files/GameWorld/media.h

## File Purpose
Header for the liquid media system (water, lava, goo, sewage, Jjaro). Defines media types, properties, and the central `media_data` struct; provides factory, query, and serialization functions for managing all fluids in a game level.

## Core Responsibilities
- Define media type constants and flags (water, lava, goo, sewage, jjaro)
- Define detonation effects and ambient/transition sounds for media interactions
- Declare the `media_data` structure (32 bytes) holding media properties: type, flags, height, current, texture, light binding
- Provide factory and update functions (`new_media`, `update_medias`)
- Query media properties: damage, sound effects, submerged rendering, danger level, environment compatibility
- Serialize/deserialize media data to/from byte streams (for save games)
- Expose XML parser integration for map loading

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `media_data` | struct | Core media instance; 32 bytes; holds type, flags, light index (for height calc), current direction/magnitude, origin, height, texture, transfer mode |
| (media types enum) | enum | `_media_water`, `_media_lava`, `_media_goo`, `_media_sewage`, `_media_jjaro` |
| (media flags enum) | enum | `_media_sound_obstructed_by_floor` ΓÇö controls sound propagation |
| (detonation types enum) | enum | Small, medium, large detonation effects, large emergence effect |
| (media sounds enum) | enum | Feet/head entering/leaving, splashing, ambient over/under, platform entering/leaving |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `MediaList` | `vector<media_data>` | global extern | All media instances in current map; size() gives max count |

## Key Functions / Methods

### new_media
- Signature: `size_t new_media(struct media_data *data)`
- Purpose: Create a new media instance in the world
- Inputs: Pointer to initialized media_data
- Outputs/Return: Index of newly created media
- Side effects: Appends to global `MediaList`
- Calls: (not inferable from header alone)
- Notes: Returns index for later reference

### update_medias
- Signature: `void update_medias(void)`
- Purpose: Update all active media each game tick (e.g., currents, height changes)
- Inputs: None (uses global `MediaList`)
- Outputs/Return: None (modifies in-place)
- Side effects: Updates media state, likely affects entity movement/physics
- Calls: (not inferable from header alone)

### get_media_data
- Signature: `media_data *get_media_data(const size_t media_index)`
- Purpose: Accessor to retrieve a single media instance by index
- Inputs: Media index (0 to MAXIMUM_MEDIAS_PER_MAP ΓêÆ 1)
- Outputs/Return: Pointer to media_data, or null for invalid index
- Side effects: None
- Notes: Added in July 2000 per comments; returns null for out-of-range

### get_media_sound
- Signature: `short get_media_sound(short media_index, short type)`
- Purpose: Retrieve sound index for a media interaction (entering, leaving, splashing, etc.)
- Inputs: Media index, sound type (one of enum media_sounds)
- Outputs/Return: Sound index to play

### get_media_damage
- Signature: `struct damage_definition *get_media_damage(short media_index, _fixed scale)`
- Purpose: Retrieve damage definition for media (e.g., lava burns, goo suffocates)
- Inputs: Media index, scale factor
- Outputs/Return: Pointer to damage_definition

### IsMediaDangerous
- Signature: `bool IsMediaDangerous(short media_type)`
- Purpose: Determine if a media type is hazardous (affects NPC pathfinding)
- Inputs: Media type enum value
- Outputs/Return: true if dangerous; false otherwise
- Notes: Added Feb 2000; prevents monsters from stepping into harmful media

### pack_media_data / unpack_media_data
- Signature: `uint8 *pack_media_data(uint8 *Stream, media_data* Objects, size_t Count)` (and unpack variant)
- Purpose: Serialize media data to/from byte streams for save games
- Inputs: Byte stream pointer, array of media objects, count
- Outputs/Return: Updated stream pointer after packing/unpacking
- Side effects: Modifies stream buffer; updates Objects array (unpack only)
- Notes: Added Aug 2000 for save/load compatibility

### Liquids_GetParser
- Signature: `XML_ElementParser *Liquids_GetParser()`
- Purpose: Provide XML parser for loading liquid definitions from map/config files
- Outputs/Return: Pointer to configured XML_ElementParser for "Liquids" element
- Notes: Added May 2000; enables data-driven media properties

## Control Flow Notes
Media system integrates into the main game tick loop via `update_medias()`, which likely runs each frame to update currents, heights, and sound ambient. The XML parser (`Liquids_GetParser`) is called during map initialization to bind media definitions from map files. Queries (`get_media_sound`, `get_media_damage`, etc.) are called reactively when entities interact with media.

## External Dependencies
- `#include <vector>` ΓÇö STL container for dynamic media list
- `#include "map.h"` ΓÇö Provides `SLOT_IS_USED` macro and world coordinate types
- `#include "XML_ElementParser.h"` ΓÇö Base class for XML parsing integration
