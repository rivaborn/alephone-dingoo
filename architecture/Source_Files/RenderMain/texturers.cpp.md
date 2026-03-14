# Source_Files/RenderMain/texturers.cpp

## File Purpose
This is a preprocessor-heavy compilation hub that generates specialized texture rendering functions by repeatedly including a template file (`low_level_textures.cpp`) with different macro configurations. It instantiates texturer code for different color depths (8, 16, 32-bit) and texture variants (normal, transparent, translucent, big textures).

## Core Responsibilities
- Provide a utility function (`NextLowerExponent`) for bit-level calculations
- Act as a template instantiation hub, configuring and including `low_level_textures.cpp` with different preprocessor defines
- Generate function variants covering:
  - 3 color depths: 8-bit, 16-bit, 32-bit
  - Transparency modes: opaque, transparent, translucent
  - Texture sizes: normal, big (larger)
  - Polygon orientations: horizontal and vertical

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### NextLowerExponent
- Signature: `inline int NextLowerExponent(int n)`
- Purpose: Computes the bit position of the most significant bit (floor(logΓéé(n)))
- Inputs: `int n` ΓÇö an integer value
- Outputs/Return: `int` ΓÇö the exponent (bit shift count)
- Side effects: None
- Calls: None
- Notes: Loop-based bit-shift algorithm. Returns 0 for n=1, 1 for n=2ΓÇô3, 2 for n=4ΓÇô7, etc.

## Control Flow Notes
The file has no runtime control flow. Instead, it uses a meta-programming pattern: it conditionally defines sets of macros and then includes `low_level_textures.cpp` multiple times. For each color depth:
- **8-bit**: 4 includes (2 transparency modes ├ù 2 texture sizes)
- **16-bit & 32-bit**: 8 includes each (2 transparency ├ù 2 texture sizes ├ù 2 translucency modes)

Total: 20 instantiations of `low_level_textures.cpp`. The macro flags (`BIT_DEPTH`, `TRANSPARENT_TEXTURERS`, `BIG_TEXTURERS`, `TRANSLUCENT_TEXTURERS`) control which code paths are compiled within the included file, effectively generating a specialized function variant for each combination.

## External Dependencies
- `#include "texturers.h"` ΓÇö declares all function prototypes and types
- `#include "low_level_textures.cpp"` ΓÇö included repeatedly; defines the actual texture rendering functions
- Declared externals (defined elsewhere):
  - `number_of_shading_tables`, `shading_table_fractional_bits`, `shading_table_size`, `texture_random_seed` ΓÇö global state from the texture/shading subsystem
