# Source_Files/RenderOther/fades.cpp

## File Purpose
Implements the screen fading and color-tinting system for the Aleph One game engine. Manages visual effects like cinematic fades, damage flashes, and environmental tints (water, lava, sewage) using both CPU color-table manipulation and GPU-accelerated OpenGL rendering.

## Core Responsibilities
- Manage fade state transitions with time-based interpolation
- Apply six distinct color blending algorithms (tint, burn, dodge, negate, randomize, soft-tint)
- Support environmental effect layering (e.g., tint blue while underwater)
- Enforce fade priority system to prevent conflicting effects
- Provide both software (color-table) and hardware (OpenGL) rendering paths
- Parse and apply XML-based fade definitions at runtime
- Implement gamma correction for display calibration

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `fade_data` | struct | Runtime state of active fade (flags, type, timing, color tables) |
| `fade_definition` | struct | Configuration for a fade type (procedure, color, transparency range, period, priority) |
| `fade_effect_definition` | struct | Maps environment effects to fade types with opacity |
| `fade_proc` | typedef | Function pointer: applies color transformation `(color_table*, color_table*, rgb_color*, _fixed)` |
| `OGL_Fader` | struct | OpenGL fader queue entry (type, 4-channel color) |
| `XML_FaderParser` | class | XML parser for fade definition elements |
| `XML_LiquidFaderParser` | class | XML parser for environment effect definitions |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `fade` | `fade_data*` | static | Global fade state; allocated once in initialize_fades() |
| `fades_random_seed` | uint16 | static | PRNG seed for randomize_color_table effect |
| `CurrentOGLFader` | `OGL_Fader*` | static | Active OpenGL fader queue entry (NULL if inactive); ifdef HAVE_OPENGL |
| `FadeEffectDelay` | int | static | Countdown timer for delayed fade-effect application (MacOS workaround) |
| `fade_definitions[]` | `fade_definition[32]` | static | Lookup table of all 32 fade types with hardcoded defaults |
| `fade_effect_definitions[]` | `fade_effect_definition[5]` | static | Environment effects (water, lava, sewage, jjaro, goo) |
| `actual_gamma_values[]` | float[8] | static | Gamma correction curves (range 0.70ΓÇô1.3) |
| `original_fade_definitions` | `fade_definition*` | static | Backup for XML override reset capability |
| `original_fade_effect_definitions` | `fade_effect_definition*` | static | Backup for XML override reset capability |

## Key Functions / Methods

### initialize_fades
- Signature: `void initialize_fades(void)`
- Purpose: Initialize fade system at engine startup
- Inputs: None
- Outputs/Return: None
- Side effects: Allocates global `fade` struct, clears flags
- Calls: `new`, `assert`
- Notes: Called once; must precede any fade operations

### update_fades
- Signature: `bool update_fades(void)`
- Purpose: Advance active fade one frame; interpolate transparency
- Inputs: None (uses global `fade` and `machine_tick_count()`)
- Outputs/Return: `true` if fade still active, `false` if finished
- Side effects: Updates `fade->flags`, calls `recalculate_and_display_color_table()`, may deactivate fade
- Calls: `FADE_IS_ACTIVE`, `get_fade_definition`, `machine_tick_count`, `recalculate_and_display_color_table`
- Notes: Handles random transparency flicker if `_random_transparency_flag` set; respects definition period

### start_fade / explicit_start_fade
- Signature: `void explicit_start_fade(short type, struct color_table *original, struct color_table *animated)`
- Purpose: Initiate a fade effect with given color tables; `start_fade()` is wrapper using global tables
- Inputs: fade type ID, source and destination color tables
- Outputs/Return: None
- Side effects: Activates fade, updates `fade->type`, `fade->last_update_tick`, stores table pointers
- Calls: `get_fade_definition`, `machine_tick_count`, `recalculate_and_display_color_table`, priority comparison
- Notes: Rejects lower-priority fades interrupting higher-priority ones; enforces `MINIMUM_FADE_RESTART` delay to prevent rapid re-trigger of same fade

### stop_fade
- Signature: `void stop_fade(void)`
- Purpose: Immediately halt current fade and apply final transparency
- Inputs: None
- Outputs/Return: None
- Side effects: Deactivates fade, applies final state instantly
- Calls: `FADE_IS_ACTIVE`, `get_fade_definition`, `recalculate_and_display_color_table`
- Notes: Used for emergency termination or fade cancellation

### set_fade_effect
- Signature: `void set_fade_effect(short type)`
- Purpose: Set environmental fade effect (water/lava tint layer)
- Inputs: effect type ID (or `NONE`)
- Outputs/Return: None
- Side effects: Updates `fade->fade_effect_type`, recalculates color table if not currently fading
- Calls: `FadeEffectDelay` countdown check, `recalculate_and_display_color_table`, `OGL_FaderActive`
- Notes: Implements MacOS workaround via `FadeEffectDelay` countdown; sets up both software and OpenGL paths

### recalculate_and_display_color_table
- Signature: `void recalculate_and_display_color_table(short type, _fixed transparency, struct color_table *original, struct color_table *animated, bool fade_active)`
- Purpose: Apply fade effect(s) to color table and render to screen/GPU
- Inputs: fade type, transparency value (0 to FIXED_ONE), source/dest color tables, fade-still-active flag
- Outputs/Return: None (modifies `animated_color_table` in-place)
- Side effects: Calls OpenGL fader setup, invokes fade procedure function, calls `animate_screen_clut`
- Calls: `SetOGLFader`, `get_fade_effect_definition`, `get_fade_definition`, fade procedure (e.g., `tint_color_table`)
- Notes: Applies environment effect layer first, then main fade layer on top for effect stacking

### Color transformation functions (tint_color_table, randomize_color_table, etc.)
- Signature (all): `static void <name>_color_table(struct color_table *original, struct color_table *animated, struct rgb_color *color, _fixed transparency)`
- Purpose: Apply a specific blending algorithm (linear tint, random noise, XOR negate, dodge, burn, soft tint)
- Inputs: source color table, tint/effect color, transparency level
- Outputs/Return: None (modifies `animated_color_table` in-place)
- Side effects: May set `CurrentOGLFader->Type` and color channels if OpenGL active
- Calls: Color manipulation macros (bit-shifts, `PIN`, `MAX`, `CEILING`, `FLOOR`); OpenGL or CPU path
- Notes: Each has fast OpenGL path (early return if `CurrentOGLFader` active); CPU path uses fixed-point arithmetic; `randomize_color_table` uses `FADES_RANDOM()` macro

### gamma_correct_color_table
- Signature: `void gamma_correct_color_table(struct color_table *uncorrected, struct color_table *corrected, short gamma_level)`
- Purpose: Apply gamma curve to color table for display calibration
- Inputs: source color table, gamma level (0ΓÇô7, where 2 is default 1.0)
- Outputs/Return: None (fills `corrected_color_table`)
- Side effects: None (pure function)
- Calls: `pow()` from `<math.h>`, `static_cast`
- Notes: Dingoo platform stub just memcpys; others apply power curve per RGB channel

### Accessors and utility functions
- **get_fade_definition(short index)**: Returns `fade_definitions[index]` with bounds checking via `GetMemberWithBounds`
- **get_fade_effect_definition(short index)**: Returns `fade_effect_definitions[index]` with bounds checking
- **get_fade_period(short type)**: Returns period field of fade definition
- **fade_finished()**: Returns `!FADE_IS_ACTIVE(fade)`
- **full_fade(short type, color_table* original)**: Synchronously execute fade to completion (loops `update_fades()`, calls `Music::instance()->Idle()` each frame)
- **SetFadeEffectDelay(int delay)**: Set `FadeEffectDelay` countdown (MacOS workaround)
- **SetOGLFader(int index)**: [ifdef HAVE_OPENGL] Set up OpenGL fader queue entry or clear if inactive
- **TranslateToOGLFader(rgb_color& color, _fixed opacity)**: Convert color and opacity to OpenGL float format

### XML_FaderParser methods
Inherited from `XML_ElementParser`; parses `<fader>` elements with attributes: `index`, `type`, `initial_opacity`, `final_opacity`, `period`, `flags`, `priority`. Backs up original definitions on first parse; restores on `ResetValues()`. Maps fader type enum to procedure function pointer.

### XML_LiquidFaderParser methods
Parses `<liquid>` elements with attributes: `index`, `fader`, `opacity`. Updates `fade_effect_definitions` array. Also supports backup/restore for reset capability.

### Faders_GetParser
- Signature: `XML_ElementParser *Faders_GetParser(void)`
- Purpose: Export root XML parser for fader configuration
- Inputs: None
- Outputs/Return: Pointer to `FadersParser` element parser
- Side effects: Attaches child parsers (`FaderParser`, `LiquidFaderParser`, color parser)
- Calls: `AddChild()` method on parser objects
- Notes: Called by XML initialization system to register fader config parser

## Control Flow Notes
**Initialization phase**: `initialize_fades()` allocates and zeros the global `fade` struct.

**Per-frame update**: `update_fades()` is called each frame; if fade active, it calculates interpolated transparency and calls `recalculate_and_display_color_table()`, which applies color transformation and sends result to `animate_screen_clut()` or OpenGL renderer.

**Fade triggering**: Code calls `start_fade(type)` or `explicit_start_fade(type, tables...)` to initiate. The priority system compares current fade priority vs. new fade; only proceeds if new fade has higher priority. `MINIMUM_FADE_RESTART` prevents rapid re-trigger of the same fade type.

**Effect layering**: If `fade->fade_effect_type != NONE` (environment effect active), `recalculate_and_display_color_table()` first applies the environment fade, then the main fade on top, enabling effects like "red flash while underwater" (cyan tint + red flash stacked).

**OpenGL optimization**: When OpenGL is active and `CurrentOGLFader` is non-NULL, color transformation functions exit early and populate the OpenGL fader queue instead of modifying CPU color tables.

## External Dependencies
- **cseries.h**: Macros (`CEILING`, `FLOOR`, `PIN`, `MAX`), memory, `GetMemberWithBounds()`, type definitions
- **fades.h**: Public API, fade/effect type enums, constants
- **screen.h**: Global `world_color_table`, `visible_color_table`; `animate_screen_clut()` function
- **ColorParser.h**: `Color_GetParser()`, `Color_SetArray()` for XML color parsing
- **OGL_Faders.h** [ifdef HAVE_OPENGL]: `OGL_Fader`, `OGL_FaderActive()`, `GetOGL_FaderQueueEntry()`
- **Music.h**: `Music::instance()->Idle()` called during `full_fade()` to keep audio responsive
- Standard library: `<string.h>`, `<stdlib.h>`, `<math.h>` (pow), `<limits.h>` (SHRT_MAX)

**Defined elsewhere**:
- `machine_tick_count()`: Get engine tick counter
- `obj_copy()`: Macro for struct copy
- `XML_ElementParser`, `ReadInt16Value()`, etc.: XML parsing framework
- `assert()`, various validation helpers from cseries
