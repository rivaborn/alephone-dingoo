# Source_Files/GameWorld/media.cpp

## File Purpose
Manages in-game media (liquids: water, lava, goo, sewage, Jjaro) for the Aleph One game engine. Handles media instance lifecycle, per-frame updates (height/texture/movement), property queries (damage, sounds, effects), and XML-driven customization of media definitions.

## Core Responsibilities
- Create, store, and retrieve media instances on the current map
- Update media states each frame (height derived from light intensity, texture, movement via current)
- Query media properties (detonation effects, ambient/event sounds, damage, submerged fade effects, texture collections)
- Parse and apply XML overrides to default media type definitions
- Serialize/deserialize media data to/from packed byte streams

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `media_data` | struct | Instance data (type, flags, light_index, current, height, texture, transfer_mode); defined in media.h |
| `media_definition` | struct | Type metadata (collection, shape, transfer_mode, damage, sounds, effects); defined elsewhere (media_definitions.h) |
| `XML_LqEffectParser` | class | Parses effect indices for media detonation events |
| `XML_LqSoundParser` | class | Parses sound indices for media events (feet/head entering/leaving, ambient) |
| `XML_LiquidParser` | class | Parses media type overrides (collection, shape, transfer mode, damage frequency, fade effect) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `MediaList` | `vector<media_data>` | global | All media instances in current map |
| `original_media_definitions` | `media_definition*` | file-static | Backup of default definitions before XML overrides |
| `LqEffectParser` | `XML_LqEffectParser` | file-static | Singleton parser for effect elements |
| `LqSoundParser` | `XML_LqSoundParser` | file-static | Singleton parser for sound elements |
| `LiquidParser` | `XML_LiquidParser` | file-static | Singleton parser for liquid elements |
| `LiquidsParser` | `XML_ElementParser` | file-static | Root parser for liquids XML tree |

## Key Functions / Methods

### get_media_data
- **Signature:** `media_data *get_media_data(const size_t media_index)`
- **Purpose:** Retrieve a media instance with bounds checking.
- **Inputs:** Index into MediaList.
- **Outputs/Return:** Pointer to media_data, or NULL if out of bounds or slot unused.
- **Side effects:** None.
- **Calls:** `GetMemberWithBounds()`, `SLOT_IS_USED()` macro.
- **Notes:** Idiot-proofs against invalid indices; NULL return indicates non-existent media.

### get_media_definition
- **Signature:** `media_definition *get_media_definition(const short type)`
- **Purpose:** Retrieve type metadata for a media type.
- **Inputs:** Media type enum (e.g., `_media_water`).
- **Outputs/Return:** Pointer to media_definition, or NULL if out of bounds.
- **Side effects:** None.
- **Calls:** `GetMemberWithBounds()`.
- **Notes:** Provides idiot-proofing via NULL on invalid type.

### new_media
- **Signature:** `size_t new_media(struct media_data *initializer)`
- **Purpose:** Allocate and initialize a new media instance in the current map.
- **Inputs:** Initializer struct with type, flags, light_index, currents, heights.
- **Outputs/Return:** Index of new media, or `UNONE` if map is full.
- **Side effects:** Modifies MediaList; calls `update_one_media()` to compute initial height/texture.
- **Calls:** `SLOT_IS_FREE()`, `MARK_SLOT_AS_USED()`, `update_one_media()`.
- **Notes:** Clears origin to (0, 0); requires light_index to be pre-loaded.

### update_medias
- **Signature:** `void update_medias(void)`
- **Purpose:** Update all active media each frame.
- **Inputs:** None (reads `dynamic_world->tick_count` for frequency).
- **Outputs/Return:** None.
- **Side effects:** Modifies height and origin of each media; applies current velocity.
- **Calls:** `update_one_media()`.
- **Notes:** Applies fractional-world-distance wrapping; uses cosine/sine tables and trig shift for velocity calculation.

### update_one_media
- **Signature:** `static void update_one_media(size_t media_index, bool force_update)`
- **Purpose:** Update a single media's height and texture.
- **Inputs:** Media index; force_update flag (currently unused).
- **Outputs/Return:** None.
- **Side effects:** Modifies mediaΓåÆheight, texture, transfer_mode.
- **Calls:** `get_media_data()`, `get_media_definition()`, `get_light_intensity()`, `BUILD_DESCRIPTOR()`.
- **Notes:** Height = low + (high ΓêÆ low) ├ù light_intensity. Disabled code suggests animated shapes were considered.

### get_media_detonation_effect
- **Signature:** `void get_media_detonation_effect(short media_index, short type, short *detonation_effect)`
- **Purpose:** Retrieve the effect ID for a media detonation event.
- **Inputs:** Media index, detonation type (e.g., `_small_media_detonation_effect`).
- **Outputs/Return:** Sets `*detonation_effect` if defined; out-param.
- **Side effects:** None.
- **Calls:** `get_media_data()`, `get_media_definition()`.
- **Notes:** Bounds-checks type; early returns on NULL media or definition.

### get_media_sound
- **Signature:** `short get_media_sound(short media_index, short type)`
- **Purpose:** Retrieve the sound ID for a media event.
- **Inputs:** Media index, sound type (e.g., `_media_snd_feet_entering`).
- **Outputs/Return:** Sound index, or `NONE` if not found or out of bounds.
- **Side effects:** None.
- **Calls:** `get_media_data()`, `get_media_definition()`.
- **Notes:** Returns NONE on invalid params.

### get_media_damage
- **Signature:** `struct damage_definition *get_media_damage(short media_index, _fixed scale)`
- **Purpose:** Get damage definition for media (e.g., lava burning).
- **Inputs:** Media index, damage scale multiplier.
- **Outputs/Return:** Pointer to damage_definition with scale set, or NULL if no damage or frequency gates it.
- **Side effects:** Modifies damageΓåÆscale.
- **Calls:** `get_media_data()`, `get_media_definition()`.
- **Notes:** Frequency gates damage per tick; gates based on `dynamic_world->tick_count` & definitionΓåÆdamage_frequency.

### IsMediaDangerous
- **Signature:** `bool IsMediaDangerous(short media_index)`
- **Purpose:** Check if a media type causes damage.
- **Inputs:** Media type index.
- **Outputs/Return:** True if base damage > 0, false otherwise.
- **Side effects:** None.
- **Calls:** `get_media_definition()`.
- **Notes:** Used to determine if monsters should avoid stepping into media.

### count_number_of_medias_used
- **Signature:** `size_t count_number_of_medias_used()`
- **Purpose:** Count highest-indexed used media slot (for save compatibility).
- **Inputs:** None.
- **Outputs/Return:** Count of used slots (or 0 if none).
- **Side effects:** None.
- **Calls:** `SLOT_IS_USED()`.
- **Notes:** Counts down from max to find last used slot; fixes countdown bug.

### unpack_media_data / pack_media_data
- **Signature:** `uint8 *unpack_media_data(uint8 *Stream, media_data* Objects, size_t Count)` / `pack` variant
- **Purpose:** Serialize/deserialize media_data arrays to/from byte streams.
- **Inputs/Outputs:** Stream pointer (advanced), object array, count.
- **Calls:** `StreamToValue()`, `ValueToStream()`.
- **Notes:** Handles all fields except padding; validates stream size matches `Count * SIZEOF_media_data`.

## Control Flow Notes
- **Initialization:** XML parsers are configured on demand via `Liquids_GetParser()`, called during map load to customize media definitions.
- **Per-frame:** `update_medias()` is called once per game tick to evolve all media heights (via light intensity) and positions (via currents).
- **Property queries:** Functions like `get_media_damage()`, `get_media_sound()` are called by gameplay code (monsters, players, effects) to determine interactions.
- **Serialization:** Pack/unpack functions serialize map state for save/load and network sync.

## External Dependencies
- **Includes:** map.h (media_data limits, macros), effects.h, fades.h, lightsource.h, SoundManager.h, DamageParser.h, Packing.h, XML_ElementParser.h.
- **Defined elsewhere:** media_definitions (array), light intensity lookup, damage types, effect/sound/fade enums, `GetMemberWithBounds()`, XML parser base class.
- **Macros used:** `SLOT_IS_USED()`, `MARK_SLOT_AS_USED()`, `CALCULATE_MEDIA_HEIGHT()`, `WORLD_FRACTIONAL_PART()`, `BUILD_DESCRIPTOR()`, trig lookup tables.
