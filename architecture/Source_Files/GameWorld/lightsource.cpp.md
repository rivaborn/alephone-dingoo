# Source_Files/GameWorld/lightsource.cpp

## File Purpose

Implements the dynamic lighting system for the Aleph One game engine (Marathon series). Manages light source creation, state transitions, intensity animation, and serialization. Lights cycle through states (becoming active ΓåÆ primary active ΓåÆ secondary active, and the inverse for inactive), with intensity computed by pluggable animation functions (constant, linear, smooth, flicker).

## Core Responsibilities

- **Light lifecycle**: Create new lights (`new_light`), track active lights in global `LightList`, support cleanup via slot marking
- **Frame updates**: Advance light phases each tick, trigger state transitions when phase overflows period, recalculate intensity
- **State management**: Transition lights between 6 states (becoming/primary/secondary ├ù active/inactive); support stateless lights that skip intermediate transitions
- **Intensity animation**: Dispatch 4 lighting functions (constant, linear, smooth with cosine taper, flicker with randomness)
- **Status control**: Query and toggle light on/off status; support bulk switching by tag; integrate with Lua script hooks
- **Serialization**: Pack/unpack light data to/from binary streams; convert Marathon 1 light format to Marathon 2+ format
- **Default configurations**: Define 3 standard light types (_normal_light, _strobe_light, _lava_light) with preset animation specs

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `light_data` | struct | Runtime light instance: current state, phase, period, intensity, linked static config |
| `static_light_data` | struct | Immutable light template: type, flags, phase offset, 6 state animation specs, tag for bulk control |
| `light_definition` | struct | Default preset: sound effects + static_light_data template for a light type |
| `lighting_function_specification` | struct | Animation spec: function index, period, delta_period, intensity, delta_intensity |
| `old_light_data` | struct | Marathon 1 light format (for migration): flags, type, mode, phase, min/max intensity, period |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `LightList` | `vector<light_data>` | global | Active lights; indexed by light_index; size = MAXIMUM_LIGHTS_PER_MAP |
| `light_definitions[NUMBER_OF_LIGHT_TYPES]` | `light_definition[]` | static | Presets for _normal_light, _strobe_light, _lava_light |
| `lighting_functions[NUMBER_OF_LIGHTING_FUNCTIONS]` | function ptr array | static | Dispatch table for 4 lighting animation functions |

## Key Functions / Methods

### new_light
- **Signature:** `short new_light(struct static_light_data *data)`
- **Purpose:** Allocate and initialize a new light instance from a static template.
- **Inputs:** `data` ΓÇö static light configuration (type, flags, animation specs, tag); NULL returns NONE.
- **Outputs/Return:** Index of newly created light (0ΓÇôMAXIMUM_LIGHTS_PER_MAPΓêÆ1), or NONE if no free slot.
- **Side effects:** Marks slot as used; initializes phase, state, intensity; triggers Lua `L_Call_Light_Activated` if HAVE_LUA; updates panel switch position.
- **Calls:** `MARK_SLOT_AS_USED`, `change_light_state`, `lighting_function_dispatch`, `get_lighting_function_specification`
- **Notes:** Initializes light to secondary state first (to set `final_intensity`), then transitions to primary state. Phase is set from `data->phase` then rephased.

### update_lights
- **Signature:** `void update_lights(void)`
- **Purpose:** Advance all active lights by one tick; recalculate intensity based on phase and lighting function.
- **Inputs:** None (operates on global `LightList` and global tick state).
- **Outputs/Return:** None (modifies in-place).
- **Side effects:** Increments `phase` for each light; calls `rephase_light` to handle state transitions; updates `intensity`.
- **Calls:** `rephase_light`, `lighting_function_dispatch`, `get_lighting_function_specification`
- **Notes:** Called once per frame during world update. Skips slots marked as unused.

### change_light_state
- **Signature:** `void change_light_state(size_t light_index, short new_state)`
- **Purpose:** Transition light to a new state; reset phase to 0 and sample period/intensity for that state.
- **Inputs:** `light_index`, `new_state` (one of 6 enum values: becoming/primary/secondary ├ù active/inactive).
- **Outputs/Return:** None (modifies light in-place).
- **Side effects:** Sets phase=0, period from spec + random delta, initial_intensity from current intensity, final_intensity from spec + random delta.
- **Calls:** `get_light_data`, `get_lighting_function_specification`, `global_random`
- **Notes:** Stores current intensity as `initial_intensity` for smooth transitions. Period randomization prevents temporal aliasing of multiple lights.

### rephase_light
- **Signature:** `static void rephase_light(short light_index)`
- **Purpose:** Handle phase overflow: when phase ΓëÑ period, loop the light to its next state.
- **Inputs:** `light_index`.
- **Outputs/Return:** None.
- **Side effects:** Repeatedly calls `change_light_state` while phase ΓëÑ period, advancing through the state machine (becoming_active ΓåÆ primary_active ΓåÆ secondary_active ΓåÆ loop or becoming_inactive, etc.). Stateless lights skip intermediate transitions.
- **Calls:** `get_light_data`, `change_light_state`
- **Notes:** Subtracts period from phase each loop; handles multi-tick overflows. Non-stateless lights cycle: primary Γåö secondary in active states; stateless lights skip to becoming_inactive/becoming_active.

### get_light_status
- **Signature:** `bool get_light_status(size_t light_index)`
- **Purpose:** Query whether light is perceptually "on" (active) or "off" (inactive).
- **Inputs:** `light_index`.
- **Outputs/Return:** `true` if state is becoming/primary/secondary_active; `false` if becoming/primary/secondary_inactive.
- **Side effects:** None.
- **Calls:** `get_light_data`
- **Notes:** Halts with error if state is invalid. Transient becoming states count as on/off respectively.

### set_light_status
- **Signature:** `bool set_light_status(size_t light_index, bool new_status)`
- **Purpose:** Toggle light on/off (if not already in target state). For non-stateless lights, transition to becoming_active/becoming_inactive.
- **Inputs:** `light_index`, `new_status` (true=on, false=off).
- **Outputs/Return:** `true` if status changed; `false` if already in target state or light is stateless.
- **Side effects:** Calls `change_light_state` to becoming_active/becoming_inactive; invokes Lua hook `L_Call_Light_Activated`; updates control panel switch assumption.
- **Calls:** `get_light_data`, `get_light_status`, `change_light_state`, `L_Call_Light_Activated`, `assume_correct_switch_position`
- **Notes:** Stateless lights (6-phase) do not transition; function returns false without state change.

### set_tagged_light_statuses
- **Signature:** `bool set_tagged_light_statuses(short tag, bool new_status)`
- **Purpose:** Bulk toggle all lights with matching tag.
- **Inputs:** `tag`, `new_status`.
- **Outputs/Return:** `true` if any light's status changed.
- **Side effects:** Iterates global `LightList`, calls `set_light_status` on matches.
- **Calls:** `set_light_status`
- **Notes:** Ignores tag == 0 (untagged).

### get_light_intensity
- **Signature:** `_fixed get_light_intensity(size_t light_index)`
- **Purpose:** Retrieve current intensity (_fixed-point; range [0, FIXED_ONE] nominally).
- **Inputs:** `light_index`.
- **Outputs/Return:** Intensity value, or 0 (blackness) if light invalid.
- **Side effects:** None.
- **Calls:** `get_light_data`

### lighting_function_dispatch
- **Signature:** `static _fixed lighting_function_dispatch(short function_index, _fixed initial_intensity, _fixed final_intensity, short phase, short period)`
- **Purpose:** Look up and invoke the appropriate lighting animation function.
- **Inputs:** Function index (0ΓÇô3), initial/final intensity, phase, period.
- **Outputs/Return:** Computed intensity at `phase` in the range [initial, final].
- **Calls:** `lighting_functions[function_index](...)`
- **Notes:** Asserts function_index is in range.

### constant_lighting_proc, linear_lighting_proc, smooth_lighting_proc, flicker_lighting_proc
- **Signature:** `static _fixed <name>(\_fixed initial_intensity, \_fixed final_intensity, short phase, short period)`
- **Purpose:** Compute intensity at a given phase in the animation cycle.
- **Outputs/Return:** 
  - Constant: always `final_intensity`.
  - Linear: `initial + (final - initial) * phase / period`.
  - Smooth: `initial + (final - initial) * (cosine_table[(phase * HALF_CIRCLE) / period + HALF_CIRCLE] + TRIG_MAGNITUDE) >> (TRIG_SHIFT + 1)` ΓÇö smooth cosine ramp.
  - Flicker: smooth intensity plus random jitter in [0, delta].
- **Notes:** Smooth uses trig table for sine/cosine taper. Flicker adds noise for visual variety.

**Packing/Unpacking Functions** (summary):
- `unpack_old_light_data`, `pack_old_light_data` ΓÇö M1 format (32 bytes each).
- `unpack_static_light_data`, `pack_static_light_data` ΓÇö static config (100 bytes each).
- `unpack_light_data`, `pack_light_data` ΓÇö runtime state (128 bytes each).
- `convert_old_light_data_to_new` ΓÇö migrate M1 lights to M2+; maps old light types to normal/strobe presets, rescales periods and intensities.

## Control Flow Notes

**Initialization:** When a map level loads, lights are created via `new_light()`, typically from saved level data unpacked via `unpack_light_data()`.

**Per-frame update:** Game calls `update_lights()` once per tick (part of `update_world()` or similar). Each light advances phase, checks for state transition, and recomputes intensity via the selected lighting function.

**State machine:** Each light cycles through up to 6 states depending on flags:
- Non-stateless: becoming_active ΓåÆ primary_active Γåö secondary_active (loops), and becoming_inactive ΓåÆ primary_inactive Γåö secondary_inactive (loops).
- Stateless (6-phase): becoming_active ΓåÆ primary_active ΓåÆ secondary_active ΓåÆ becoming_inactive ΓåÆ primary_inactive ΓåÆ secondary_inactive (loops).

**Triggering:** Level geometry (polygons, sides) can trigger lights via `set_light_status()` (usually invoked from control panel interaction or Pfhortran script). Lua callbacks are fired when lights activate.

## External Dependencies

- **cseries.h** ΓÇö `vhalt`, `csprintf`, `temporary`, assertion macros, common types.
- **map.h** ΓÇö `TICKS_PER_SECOND`, `MAXIMUM_LIGHTS_PER_MAP`, `LIGHT_IS_INITIALLY_ACTIVE`, `SLOT_IS_USED/FREE/MARK_*` macros, world definitions, `assume_correct_switch_position`.
- **lightsource.h** ΓÇö Type definitions (`light_data`, `static_light_data`, `lighting_function_specification`, etc.), enum constants, global `LightList`.
- **Packing.h** ΓÇö `StreamToValue`, `ValueToStream` for binary serialization.
- **lua_script.h** ΓÇö `L_Call_Light_Activated()` callback (only when `HAVE_LUA` is defined).
- **Global functions (defined elsewhere):** `global_random()`, `cosine_table[]`, `GetMemberWithBounds()`, `obj_copy()`, `SET_FLAG`, `TEST_FLAG16`, `SET_FLAG16`.
