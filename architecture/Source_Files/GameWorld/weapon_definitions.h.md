# Source_Files/GameWorld/weapon_definitions.h

## File Purpose
Defines weapon system data structures and configuration for the Aleph One game engine (Marathon). Specifies weapon mechanics, animations, ammunition, audio, physics (recoil, projectiles), and shell casing behavior. Contains hardcoded definitions for 10 weapons and serialization helpers.

## Core Responsibilities
- Define weapon classes (melee, single-trigger, dual-function, dual-wield, multipurpose)
- Define weapon behavior flags (automatic, overload, phase-firing, underwater-capable, ammo-sharing, etc.)
- Configure weapon animations (idle, firing, reloading shapes and timing)
- Specify trigger mechanics per weapon (ammo type, fire rate, charging, burst count, recoil)
- Define shell casing physics and rendering (initial position, velocity, acceleration)
- Store weapon ordering/slot assignments
- Provide pack/unpack serialization for weapon data

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `shell_casing_definition` | struct | Physics & rendering for ejected shell casings (position, velocity, acceleration) |
| `trigger_definition` | struct | Single trigger mechanics: ammo type, fire rate, recoil, projectile, sounds, burst behavior |
| `weapon_definition` | struct | Complete weapon config: class, flags, animations, visual effects, timing, two triggers |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `shell_casing_definitions[]` | static array of `shell_casing_definition` | static | Physics data for 5 shell casing types (assault rifle, pistol variants, SMG) |
| `weapon_definitions[]` | static array of `weapon_definition` | static | Runtime weapon configs (initialized from `original_weapon_definitions`) |
| `original_weapon_definitions[]` | const array of `weapon_definition` | global | Hardcoded definitions for 10 weapons (fist, magnum, plasma pistol, assault rifle, rocket, flamethrower, alien shotgun, shotgun, ball, SMG) |
| `weapon_ordering_array[]` | static array of int16 | static | Weapon slot order mapping |

## Key Functions / Methods

### unpack_weapon_definition
- **Signature:** `uint8 *unpack_weapon_definition(uint8 *Stream, weapon_definition *Objects, size_t Count)`
- **Purpose:** Deserialize weapon definitions from binary stream (save/load support)
- **Inputs:** Binary stream pointer, destination weapon array, count
- **Outputs/Return:** Advanced stream pointer
- **Side effects:** Modifies weapon definition array in-place
- **Calls:** Not defined in this file (defined elsewhere)

### pack_weapon_definition
- **Signature:** `uint8 *pack_weapon_definition(uint8 *Stream, weapon_definition *Objects, size_t Count)`
- **Purpose:** Serialize weapon definitions to binary stream (save/load support)
- **Inputs:** Binary stream pointer, source weapon array, count
- **Outputs/Return:** Advanced stream pointer
- **Side effects:** Writes to stream
- **Calls:** Not defined in this file (defined elsewhere)

## Control Flow Notes
Data-definition file with no executable logic. Weapon definitions are loaded at engine initialization and consulted during: weapon firing (ammo/timing/projectile lookup), animation sequencing, recoil application, shell casing spawning, audio triggering. Serialization functions indicate save/load integration for persistent game state.

## External Dependencies
- **Types (defined elsewhere):** `_fixed` (fixed-point number), `world_distance` (coordinate type), `uint8`
- **Constants:** `FIXED_ONE`, `FIXED_ONE_HALF`, `TICKS_PER_SECOND`, `WORLD_ONE_FOURTH`, `NONE`
- **External IDs referenced:** weapon item types (`_i_*`), projectile types (`_projectile_*`), sound IDs (`_snd_*`), collection IDs (`_collection_*`, `_weapon_in_hand_collection`)
- **Defines:** `NUMBER_OF_TRIGGERS` (inferred as 2 from trigger array usage), `NUMBER_OF_SHELL_CASING_TYPES` (5), `NUMBER_OF_WEAPONS` (10)
