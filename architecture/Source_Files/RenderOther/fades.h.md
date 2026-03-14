# Source_Files/RenderOther/fades.h

## File Purpose
Header file defining the fade and screen effect system for the Aleph One game engine. Manages cinematic fades, visual feedback effects (damage flashes, item pickups, teleports), and environmental color tints, with support for gamma correction and XML-based configuration.

## Core Responsibilities
- Define fade type enumeration (cinematic, damage, environmental tints, etc.)
- Declare fade lifecycle functions (initialize, update, start, stop, check completion)
- Provide fade effect configuration and timing queries
- Support color table manipulation and gamma correction
- Expose XML parser interface for fade configuration
- Workaround for MacOS-specific screen painting bugs via effect delay

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `_start_cinematic_fade_in` to `_fade_tint_jjaro` (enum) | enum constants | 30+ fade type identifiers covering cinematics, damage, environmental effects |
| `_effect_under_water` to `_effect_under_goo` (enum) | enum constants | 5 environmental effect types (water, lava, sewage, JjaroGoo, goo) |
| `_tint_fader_type` to `_soft_tint_fader_type` (enum) | enum constants | 6 fader function/algorithm types for color manipulation |

## Global / File-Static State
None.

## Key Functions / Methods

### initialize_fades
- Signature: `void initialize_fades(void)`
- Purpose: Initialize the fade system at engine startup
- Inputs: None
- Outputs/Return: None
- Side effects: Allocates/initializes internal fade state
- Calls: (Implementation in FADES.C)
- Notes: Must be called before any fade operations

### update_fades
- Signature: `bool update_fades(void)`
- Purpose: Advance fade animation state each frame
- Inputs: None
- Outputs/Return: Boolean (true if fade is still active)
- Side effects: Updates internal fade timing/animation state
- Calls: (Implementation in FADES.C)
- Notes: Called once per frame during main loop

### start_fade
- Signature: `void start_fade(short type)`
- Purpose: Trigger a fade effect by type
- Inputs: `type` ΓÇö fade type enum value
- Outputs/Return: None
- Side effects: Begins fade animation immediately
- Calls: (Implementation in FADES.C)

### set_fade_effect
- Signature: `void set_fade_effect(short type)`
- Purpose: Apply environmental tint or effect overlay
- Inputs: `type` ΓÇö effect type enum value
- Outputs/Return: None
- Side effects: Subject to `SetFadeEffectDelay()` workaround
- Notes: Delayed execution to work around MacOS painting bugs

### gamma_correct_color_table
- Signature: `void gamma_correct_color_table(struct color_table *uncorrected_color_table, struct color_table *corrected_color_table, short gamma_level)`
- Purpose: Apply gamma correction to color palette
- Inputs: `uncorrected_color_table`, `gamma_level` (0ΓÇô7)
- Outputs/Return: `corrected_color_table` (filled with corrected values)
- Side effects: None (pure computation)
- Notes: 8 gamma levels defined; level 2 is default

### SetFadeEffectDelay
- Signature: `void SetFadeEffectDelay(int _FadeEffectDelay)`
- Purpose: Configure delay for fade effect application
- Inputs: Delay count (number of `set_fade_effect()` calls to skip)
- Outputs/Return: None
- Side effects: Modifies internal delay counter
- Notes: MacOS-specific workaround; defers effect until dialog boxes clear

### Faders_GetParser
- Signature: `XML_ElementParser *Faders_GetParser()`
- Purpose: Retrieve XML parser for fade configuration
- Inputs: None
- Outputs/Return: Pointer to XML element parser object
- Side effects: None
- Notes: Enables XML-based fade customization

**Trivial helpers** (documented in notes): `stop_fade()`, `fade_finished()`, `get_fade_period()`, `explicit_start_fade()`, `full_fade()` ΓÇö straightforward wrappers for fade state queries and specialized fade initiation.

## Control Flow Notes
This module fits into the **frame/update** phase:
- `initialize_fades()` runs during engine initialization
- `update_fades()` is called once per rendered frame to animate active fade effects
- `start_fade()`, `set_fade_effect()` are called reactively (e.g., when player takes damage, picks up item, or enters water)
- `stop_fade()` and `fade_finished()` are used to query/clear fade state between frames

## External Dependencies
- **`"XML_ElementParser.h"`** ΓÇö XML parsing framework for configuration; provides `XML_ElementParser` base class used by `Faders_GetParser()`
- **`color_table` struct** ΓÇö Defined elsewhere; used for gamma correction and color palette management
- Standard C library (stdio implicit in included headers)
