# Source_Files/shell_misc.cpp

## File Purpose
Implements cheat code functionality for the Aleph One engine, including detection, XML configuration, and application of cheat effects. Also provides miscellaneous shell operations: periodic idle processing, memory management for level transitions, and integration with the game's UI and audio systems.

## Core Responsibilities
- Detect and process cheat code keywords from player input
- Apply cheat effects (health boost, weapons, invincibility, map reveal, etc.)
- Parse and configure cheats via XML/MML files
- Execute periodic background tasks (music, network, sound updates)
- Manage memory allocation with fallback strategies during level loads
- Track global cheat state and input mode flags

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `keyword_data` | struct | Maps cheat tag enum to uppercase keyword string |
| `XML_CheatsParser` | class | XML parser for root `<cheats>` element; reads "on" and "mac_keymod" attributes |
| `XML_CheatKeywordParser` | class | XML parser for customizable `<keyword>` elements; allows remapping cheat strings via MML |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `CheatsActive` | bool | global | Master flag enabling/disabling all cheat functionality |
| `chat_input_mode` | bool | global | Whether player is in chat text-entry mode |
| `CheatCodeModMask` | unsigned short | static (Mac) | Keyboard modifier required to enter cheats on Mac |
| `keyword_buffer` | char[21] | static | 20-char sliding window of typed chars; detects cheat keywords |
| `keywords` | keyword_data[] | static | Array of ~20 cheat definitions (tag + keyword string) |
| `CheatKeywordParser` | XML_CheatKeywordParser | static | Singleton parser for keyword XML elements |
| `CheatsParser` | XML_CheatsParser | static | Singleton parser for cheats root element |

## Key Functions / Methods

### handle_keyword
- Signature: `void handle_keyword(int tag)`
- Purpose: Apply the game effect for a detected cheat code
- Inputs: `tag` - cheat enum (_tag_health, _tag_invincible, _tag_save, etc.)
- Outputs/Return: None
- Side effects: Modifies player state (energy, oxygen, items, powerups); marks UI dirty; may save game
- Calls: `try_and_add_player_item()`, `process_player_powerup()`, `AddItemsToPlayer()`, `get_item_kind()`, `save_game()`, `update_interface()`, `mark_shield_display_as_dirty()`, `mark_oxygen_display_as_dirty()`, `accelerate_monster()`
- Notes: Implements 20+ cheat effects; health cheat has cascading stages (1x, 2x, 3x max); "ASLAG" cheat grants all weapons/ammo

### process_keyword_key
- Signature: `int process_keyword_key(char key)`
- Purpose: Detect cheat codes by matching typed character against keyword buffer
- Inputs: `key` - single character (converted to uppercase)
- Outputs/Return: Tag enum if keyword matched, else NONE
- Side effects: Shifts keyword_buffer left, inserts char at end, clears buffer on match
- Calls: `strcmp()`, `memset()`
- Notes: Sliding 20-char window; checks all keywords; clears on match to prevent re-triggering

### global_idle_proc
- Signature: `void global_idle_proc(void)`
- Purpose: Periodic background task processing during event loops
- Inputs: None
- Outputs/Return: None
- Side effects: Updates music playback, network I/O, sound manager state
- Calls: `Music::instance()->Idle()`, `network_speaker_idle_proc()`, `network_microphone_idle_proc()`, `SoundManager::instance()->Idle()`
- Notes: Networking calls compile out if DISABLE_NETWORKING defined

### level_transition_malloc
- Signature: `void *level_transition_malloc(size_t size)`
- Purpose: Allocate memory during level transitions with aggressive fallback strategies
- Inputs: `size` - bytes to allocate
- Outputs/Return: Valid pointer (never NULL)
- Side effects: On allocation failure, unloads sounds, then collections, then retries
- Calls: `malloc()`, `SoundManager::instance()->UnloadAllSounds()`, `unload_all_collections()`
- Notes: 3-stage fallback; ensures level load succeeds even under memory pressure

### free_and_unlock_memory
- Signature: `void free_and_unlock_memory(void)`
- Purpose: Release temporary memory when critical operations need headroom
- Inputs: None
- Outputs/Return: None
- Side effects: Stops all sounds; purges/compacts memory (Mac-specific)
- Calls: `SoundManager::instance()->StopAllSounds()`, `PurgeMem()`, `CompactMem()`

### Cheats_GetParser
- Signature: `XML_ElementParser *Cheats_GetParser()`
- Purpose: Provide root XML parser for cheat configuration; attach child keyword parser
- Inputs: None
- Outputs/Return: Pointer to CheatsParser singleton
- Side effects: Adds CheatKeywordParser as child on initialization
- Notes: Called during engine setup to integrate cheats into MML parsing tree

### XML_CheatsParser::HandleAttribute
- Signature: `bool XML_CheatsParser::HandleAttribute(const char *Tag, const char *Value)`
- Purpose: Parse `<cheats>` element attributes
- Inputs: `Tag` - "on" or "mac_keymod"; `Value` - attribute value string
- Outputs/Return: true on recognized attribute
- Side effects: Sets `CheatsActive` or `CheatCodeModMask` (Mac-only)

### XML_CheatKeywordParser methods
- `Start()`: Initialize; set `IsPresent = false`
- `HandleAttribute()`: Parse "index" attribute; validate bounds
- `AttributesDone()`: Verify "index" was required and present
- `HandleString()`: Decode UTF-8 text into keywords array at Index; uppercase conversion

## Control Flow Notes
- **Init**: `Cheats_GetParser()` called at engine startup; XML/MML parsed; `CheatsActive` enabled if configured
- **Per-frame**: `global_idle_proc()` runs regularly to service music, network, sound
- **Input**: Per-keystroke, `process_keyword_key()` checks buffer; on match, `handle_keyword()` applies effect
- **Memory**: `level_transition_malloc()` called for level-load allocations with staged unload fallbacks

## External Dependencies
- **Player/world state**: `local_player`, `local_player_index`, `dynamic_world` (defined in player.h/world.h/map.h)
- **Item/powerup**: `try_and_add_player_item()`, `process_player_powerup()`, `process_new_item_for_reloading()`, `get_item_kind()`
- **UI updates**: `mark_shield_display_as_dirty()`, `mark_oxygen_display_as_dirty()`, `update_interface()`
- **Physics**: `accelerate_monster()`
- **Save/load**: `save_game()`, `unload_all_collections()`
- **Audio**: `Music::instance()`, `SoundManager::instance()` (singletons)
- **Network**: `network_speaker_idle_proc()`, `network_microphone_idle_proc()`
- **XML parsing**: `ReadBooleanValueAsBool()`, `ReadUInt16Value()`, `ReadBoundedInt16Value()`, `DeUTF8_C()`, `StringsEqual()`, `UnrecognizedTag()`
- **Standard**: `<ctype.h>` (`toupper`), string operations (`strcmp`, `memset`)
