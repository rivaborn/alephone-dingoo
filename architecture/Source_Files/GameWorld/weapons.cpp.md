# Source_Files/GameWorld/weapons.cpp

## File Purpose
Core weapon system implementation for the Aleph One engine (Marathon). Manages player weapon state machines, firing/reloading cycles, ammunition tracking, visual animations (including shell casings), and weapon-to-environment compatibility. Handles both per-trigger dual-wield mechanics and weapon switching.

## Core Responsibilities
- Weapon state machine updates (idle, firing, reloading, recovering, charging, etc.)
- Trigger (primary/secondary) management and action polling
- Ammunition tracking, loading, and depletion
- Weapon raising/lowering animations and sequences
- Shell casing visual effect spawning and updating
- Fired projectile creation and ballistic calculations
- Weapon readiness and environment compatibility checks
- Player item pickup processing for ammo and weapon acquisition
- Game state serialization/deserialization (packing/unpacking)
- XML configuration parsing for weapon definitions and shell casings

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `player_weapon_data` | struct | Holds all weapon state for one player: current weapon, desired weapon, array of weapons, shell casings |
| `weapon_data` | struct | Per-weapon state: weapon type, flags, array of trigger_data |
| `trigger_data` | struct | Per-trigger state: animation phase, rounds loaded, shots fired/hit, firing duration |
| `shell_casing_data` | struct | Shell casing visual: type, frame, position, velocity |
| `weapon_definition` | struct (external) | Loaded weapon config: class, timing, shapes, light, ammunition types |
| `trigger_definition` | struct (external) | Loaded trigger config: magazine size, ammo type, sound, projectile type, recoil |
| `shell_casing_definition` | struct (external) | Loaded shell casing config: visuals, initial/delta velocity |
| `weapon_display_information` | struct (external) | For rendering: collection, shape, position, frame animation data |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `player_weapons_array` | `struct player_weapon_data *` | static | Array of weapon state for all players, allocated once at init |
| `original_shell_casing_definitions` | `struct shell_casing_definition *` | static | Backup of shell casing defs for XML reset |
| `original_weapon_ordering_array` | `int16 *` | static | Backup of weapon power ordering for XML reset |
| `ShellCasingParser` | `XML_ShellCasingParser` | static | XML element parser for shell casing config |
| `WeaponOrderParser` | `XML_WeaponOrderParser` | static | XML element parser for weapon order config |
| `WeaponsParser` | `XML_ElementParser` | static | Root XML element parser for weapons subtree |

## Key Functions / Methods

### initialize_weapon_manager
- **Signature:** `void initialize_weapon_manager(void)`
- **Purpose:** Engine startup initialization; allocates and zeros the global player weapon array.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Allocates `MAXIMUM_NUMBER_OF_PLAYERS * sizeof(player_weapon_data)` bytes globally
- **Calls:** `malloc()`, `objlist_clear()`
- **Notes:** Called once at engine startup before any players exist.

### initialize_player_weapons_for_new_game
- **Signature:** `void initialize_player_weapons_for_new_game(short player_index)`
- **Purpose:** Full weapon reset for new game; clears all weapon state and shell casings.
- **Inputs:** `player_index` ΓÇô player ID [0, max_players)
- **Outputs/Return:** None
- **Side effects:** Zeroes weapon state; calls `initialize_shell_casings()`
- **Calls:** `get_player_weapon_data()`, `obj_clear()`, `initialize_player_weapons()`, `initialize_shell_casings()`
- **Notes:** Called when player starts a fresh game session.

### initialize_player_weapons
- **Signature:** `void initialize_player_weapons(short player_index)`
- **Purpose:** Per-player weapon initialization; sets all weapon types, resets current/desired weapons, zeroes ammo and triggers.
- **Inputs:** `player_index`
- **Outputs/Return:** None
- **Side effects:** Writes to player weapon array; sets player light intensity to natural level
- **Calls:** `get_player_weapon_data()`, `get_player_data()`, `reset_trigger_data()`
- **Notes:** Called after player creation or when leveling up.

### update_player_weapons
- **Signature:** `void update_player_weapons(short player_index, uint32 action_flags)`
- **Purpose:** Per-frame weapon state machine; processes trigger input, manages firing/reloading, updates animations and light.
- **Inputs:** `player_index`, `action_flags` (bit flags for trigger state, cycling, etc.)
- **Outputs/Return:** None
- **Side effects:** Updates trigger states, fires projectiles, plays sounds, updates shell casings, modifies weapon light intensity decay
- **Calls:** `update_shell_casings()`, `test_raise_double_weapon()`, `get_active_trigger_count_and_states()`, `update_sequence()`, `handle_trigger_down()`, `handle_trigger_up()`, state machine via calls to `fire_weapon()`, `reload_weapon()`, `put_rounds_into_weapon()`
- **Notes:** Complex state machine driving weapon animations and actions; called once per tick per player. Weapon light is additive (not replacing) player's natural light.

### mark_weapon_collections
- **Signature:** `void mark_weapon_collections(bool loading)`
- **Purpose:** Mark weapon and projectile graphics collections for loading/unloading during level transitions.
- **Inputs:** `loading` ΓÇô true to load, false to unload
- **Outputs/Return:** None
- **Side effects:** Calls mark functions on shape collections and projectile collections
- **Calls:** `get_weapon_definition()`, `mark_collection_for_loading/unloading()`, `mark_projectile_collections()`
- **Notes:** Iterates all weapons; skips projectile marking for ball weapon.

### process_new_item_for_reloading
- **Signature:** `void process_new_item_for_reloading(short player_index, short item_type)`
- **Purpose:** Handle player pickup of weapons or ammunition; reload/refill the appropriate weapon trigger(s).
- **Inputs:** `player_index`, `item_type` (from items.h enum)
- **Outputs/Return:** None
- **Side effects:** Resets trigger ammo; may call `ready_weapon()` to switch to picked-up weapon; marks weapon display dirty
- **Calls:** `get_item_kind()`, `get_weapon_definition()`, `get_trigger_definition()`, `get_player_data()`, `get_player_weapon_data()`, `reset_trigger_data()`, `should_switch_to_weapon()`, `ready_weapon()`, `get_active_trigger_count_and_states()`
- **Notes:** Handles both weapon pickups (with ammo loading) and ammo-only pickups. Complex branching for twofisted pistols vs. single weapons.

### check_player_weapons_for_environment_change
- **Signature:** `void check_player_weapons_for_environment_change(void)`
- **Purpose:** Verify current weapon still works in new environment; switch to fallback if not.
- **Inputs:** None (uses `dynamic_world`)
- **Outputs/Return:** None
- **Side effects:** May call `select_next_best_weapon()` to switch weapons; calls `calculate_ticks_from_shapes()`
- **Calls:** `player_has_valid_weapon()`, `get_player_weapon_data()`, `weapon_works_in_current_environment()`, `select_next_best_weapon()`, `calculate_ticks_from_shapes()`
- **Notes:** Called when entering a level; prevents vacuum/magnetic weapons from firing in incompatible environments.

### discharge_charged_weapons
- **Signature:** `void discharge_charged_weapons(short player_index)`
- **Purpose:** Fire any fully-charged weapons when player dies (drops charged projectiles).
- **Inputs:** `player_index`
- **Outputs/Return:** None
- **Side effects:** Calls `fire_weapon()` for charged triggers
- **Calls:** `player_has_valid_weapon()`, `get_player_weapon_data()`, `get_active_trigger_count_and_states()`, `get_player_trigger_data()`, `fire_weapon()`
- **Notes:** Ensures charged energy isn't lost on death; fires at max charge.

### player_hit_target
- **Signature:** `void player_hit_target(short player_index, short weapon_identifier)`
- **Purpose:** Record hit statistics when a projectile fired by this weapon hits something.
- **Inputs:** `player_index`, `weapon_identifier` (encoded weapon + trigger)
- **Outputs/Return:** None
- **Side effects:** Increments `shots_hit` counter for the trigger
- **Calls:** Decoding macros `GET_WEAPON_FROM_IDENTIFIER()`, `GET_TRIGGER_FROM_IDENTIFIER()`
- **Notes:** Called from projectile system after impact detection.

### Shell Casing Functions (4 functions)
**initialize_shell_casings, new_shell_casing, update_shell_casings, get_shell_casing_display_data**
- Collectively manage shell casing visual effects (ejected cartridges).
- `initialize_shell_casings`: Mark all slots as free.
- `new_shell_casing`: Allocate a new casing, init position/velocity from definition, add randomness.
- `update_shell_casings`: Apply physics (velocity + gravity) each tick, remove when off-screen.
- `get_shell_casing_display_data`: Provide animation frame and position for rendering; advance frame counter.

### Packing/Unpacking Functions (6 inline + 2 main)
**StreamToTrigData, TrigDataToStream, StreamToWeapData, WeapDataToStream, StreamToShellData, ShellDataToStream, unpack_player_weapon_data, pack_player_weapon_data, unpack_weapon_definition, pack_weapon_definition**
- Serialize/deserialize weapon state to/from binary streams.
- Used for save game, network transmission, or state snapshots.
- Each struct has paired stream functions that read/write fields in fixed order.

### fire_weapon
- **Signature:** `static void fire_weapon(short player_index, short which_trigger, _fixed charged_amount, bool flail_wildly)`
- **Purpose:** Internal function to actually fire a weapon trigger; create projectile, play sound, apply recoil, set animation state.
- **Inputs:** `player_index`, `which_trigger` (primary/secondary), `charged_amount` ([0, FIXED_ONE] for charged weapons), `flail_wildly` (ignore aiming on death)
- **Outputs/Return:** None
- **Side effects:** Creates projectile, plays firing sound, updates trigger state, calculates origin/vector, applies muzzle flash light
- **Calls:** `get_player_weapon_data()`, `get_current_weapon_definition()`, `get_player_trigger_data()`, `calculate_weapon_origin_and_vector()`, `new_projectile()`, `play_weapon_sound()`, `get_trigger_definition()`
- **Notes:** Not inferable from truncated file, but called from state machine and `discharge_charged_weapons()`.

### reload_weapon
- **Signature:** `static bool reload_weapon(short player_index, short which_trigger)`
- **Purpose:** Check ammunition and initiate reload sequence if ammo available.
- **Inputs:** `player_index`, `which_trigger`
- **Outputs/Return:** true if reload initiated, false if no ammo
- **Side effects:** Transitions trigger state to `_weapon_awaiting_reload` if ammo exists
- **Calls:** `get_player_trigger_data()`, `player_weapon_has_ammo()`, `get_player_trigger_definition()`
- **Notes:** Not fully visible in truncated file; called from state machine.

### XML Parsers (2 classes: XML_ShellCasingParser, XML_WeaponOrderParser)
- Parse shell casing and weapon order configuration from game XML files.
- Support attribute parsing, validation, and reset-to-defaults.
- Allow modders to customize shell casings and weapon cycling order.

## Control Flow Notes

**Initialization phase:**
- `initialize_weapon_manager()` called once at engine startup
- `initialize_player_weapons_for_new_game()` called per player at game start
- `initialize_player_weapons()` called per player setup
- `check_player_weapons_for_environment_change()` called when entering a level

**Per-frame update phase:**
- `update_player_weapons()` called once per tick per player with action_flags from input queue
- Drives weapon state machine: transitions between idle, firing, reloading, charging, recovering states
- `update_sequence()` advances animation frame
- `update_shell_casings()` advances shell casing physics
- Fires projectiles via `fire_weapon()` when transitioning out of charging state
- Plays sounds and updates weapon light intensity

**Event-driven:**
- `process_new_item_for_reloading()` called when player picks up items
- `discharge_charged_weapons()` called when player dies
- `player_hit_target()` called from projectile system on hit
- Packing/unpacking called for save/load/networking

**Level transitions:**
- `check_player_weapons_for_environment_change()` verifies weapon compatibility
- `mark_weapon_collections()` manages graphics loading

No explicit shutdown phase visible; assumes system cleanup at engine exit.

## External Dependencies
- **Core engine:** `map.h` (world geometry), `projectiles.h` (fire projectile), `player.h` (player state), `interface.h` (collections, shapes), `items.h` (item types), `monsters.h` (monster references), `game_window.h` (UI updates)
- **Audio:** `SoundManager.h` (play weapon/shell casing sounds)
- **Serialization:** `Packing.h` (stream macros)
- **Configuration:** `weapon_definitions.h` (weapon/trigger/shell casing definitions, loaded from external config/XML), `XML_ElementParser.h` (XML parsing base class)
- **Standard library:** `<string.h>`, `<stdlib.h>`, `<limits.h>` (for SHRT_MAX, SHRT_MIN)
- **Platform:** `#pragma segment weapons` for 68k Macintosh (legacy)

**Defined elsewhere (referenced but not defined here):**
- `weapon_definitions`, `shell_casing_definitions`, `weapon_ordering_array` (external definition arrays)
- `dynamic_world`, `current_player_index` (global game state)
- `get_dynamic_limit()`, `get_player_data()`, `get_item_kind()` (engine accessors)
- `GetMemberWithBounds()` (bounds-checked array access macro)
