# Source_Files/GameWorld/world.cpp

## File Purpose

Core geometric and mathematical engine for Aleph One's game world. Implements trigonometric lookups, 2D/3D point transformations, distance calculations, and random number generation. Includes specialized long-distance coordinate overflow handling for large map coordinates.

## Core Responsibilities

- Initialize and provide access to precomputed sine/cosine/tangent lookup tables
- Translate and rotate 2D/3D points in world space around origins
- Calculate arctangent from Cartesian coordinates using binary search
- Compute Euclidean and approximate distances between points
- Provide deterministic random number generation with global and local seeds
- Handle integer square root and overflow-extended coordinate arithmetic

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `angle` | typedef | 9-bit angle representation (0ΓÇô511 Γëê 0ΓÇô2╧Ç) |
| `world_distance` | typedef | int16 fixed-point world coordinates |
| `world_point2d` | struct | 2D world position (x, y) |
| `world_point3d` | struct | 3D world position (x, y, z) |
| `long_vector2d` | struct | 32-bit intermediate for overflow handling |
| `long_vector3d` | struct | 32-bit intermediate for 3D overflow handling |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `cosine_table` | int16* | global | Precomputed cosine lookup (512 entries) |
| `sine_table` | int16* | global | Precomputed sine lookup (512 entries) |
| `tangent_table` | int32* | static | Precomputed tangent lookup (internal arctangent use) |
| `random_seed` | uint16 | static | Global LCG state for `global_random()` |
| `local_random_seed` | uint16 | static | Local LCG state for `local_random()` |

## Key Functions / Methods

### build_trig_tables()
- **Signature:** `void build_trig_tables(void)`
- **Purpose:** Allocate and initialize sine, cosine, and tangent lookup tables at startup
- **Inputs:** None
- **Outputs/Return:** None (modifies global pointers)
- **Side effects:** Malloc + populates `sine_table`, `cosine_table`, `tangent_table`
- **Calls:** `malloc()`, `cos()`, `sin()`, `atan()`
- **Notes:** Enforces cardinal directions (0┬░, 90┬░, 180┬░, 270┬░) exactly; tangent undefined at poles uses `INT32_MIN`

### translate_point2d()
- **Signature:** `world_point2d *translate_point2d(world_point2d *point, world_distance distance, angle theta)`
- **Purpose:** Move a 2D point by distance along direction theta
- **Inputs:** Point (modified in-place), distance, angle (normalized)
- **Outputs/Return:** Pointer to modified point
- **Side effects:** Modifies point in-place
- **Calls:** `normalize_angle()`
- **Notes:** Uses lookup table shifts; small precision loss (~1/1024) acceptable; mutates input

### translate_point3d()
- **Signature:** `world_point3d *translate_point3d(world_point3d *point, world_distance distance, angle theta, angle phi)`
- **Purpose:** Move a 3D point by distance along two spherical angles (azimuth, elevation)
- **Inputs:** Point, distance, theta (azimuth), phi (elevation)
- **Outputs/Return:** Pointer to modified point
- **Side effects:** Mutates input point
- **Calls:** `normalize_angle()` (2x)
- **Notes:** Applies phi first to scale xy distance, then theta for xy decomposition

### rotate_point2d()
- **Signature:** `world_point2d *rotate_point2d(world_point2d *point, world_point2d *origin, angle theta)`
- **Purpose:** Rotate point around origin by angle theta
- **Inputs:** Point, origin, angle
- **Outputs/Return:** Pointer to rotated point
- **Side effects:** Mutates point in-place; uses int32 intermediates for precision
- **Calls:** `normalize_angle()`
- **Notes:** Uses long temporaries to avoid loss of precision during rotation matrix ops

### transform_point2d() / transform_point3d()
- **Signature:** `world_point2d/3d *transform_point2d/3d(..., origin, theta [, phi])`
- **Purpose:** Rotate point relative to origin, returning rotated coordinates (not absolute position)
- **Inputs:** Point, origin, angles
- **Outputs/Return:** Transformed coordinates as if origin were at (0,0)
- **Side effects:** Mutates input; uses long intermediates
- **Calls:** `normalize_angle()`
- **Notes:** 3D version applies theta (xy plane) then phi (xz plane) in sequence; skips phi rotation if phi==0

### arctangent()
- **Signature:** `angle arctangent(int32 x, int32 y)`
- **Purpose:** Compute angle from Cartesian coordinates using binary search over tangent table
- **Inputs:** x, y (int32 to support long-distance)
- **Outputs/Return:** angle in [0, NUMBER_OF_ANGLES)
- **Side effects:** None (read-only tangent table)
- **Calls:** None visible
- **Notes:** Quadrant reduction (2nd/3rdΓåÆ1st/4th, 4thΓåÆ1st), octant selection, binary search on tangent table; handles (0,0) ΓåÆ 0; sign-extends negatives correctly

### distance2d() / distance3d()
- **Signature:** `world_distance distance2d/3d(world_point2d/3d *p0, world_point2d/3d *p1)`
- **Purpose:** Compute Euclidean distance; clamped to int16 max
- **Inputs:** Two points
- **Outputs/Return:** world_distance (int16), capped at INT16_MAX
- **Side effects:** None
- **Calls:** `isqrt()`
- **Notes:** Uses int32 intermediates for squared distance; distance>INT16_MAX returns INT16_MAX

### guess_distance2d()
- **Signature:** `world_distance guess_distance2d(world_point2d *p0, world_point2d *p1)`
- **Purpose:** Fast approximate distance (no sqrt; Manhattan-ish hybrid)
- **Inputs:** Two 2D points
- **Outputs/Return:** Rough distance estimate
- **Side effects:** None
- **Calls:** `GUESS_HYPOTENUSE` macro
- **Notes:** Returns `max(dx, dy) + 0.5*min(dx, dy)` for quick range checks

### isqrt()
- **Signature:** `int32 isqrt(register uint32 x)`
- **Purpose:** Integer square root via binary digit-by-digit extraction
- **Inputs:** Unsigned 32-bit integer
- **Outputs/Return:** Floor of square root; rounds up if remainder > result
- **Side effects:** None
- **Calls:** None
- **Notes:** Single-pass loop; highly optimized bit-shift arithmetic; detailed multi-paragraph comment explains algorithm

### global_random() / local_random()
- **Signature:** `uint16 global_random(void) / uint16 local_random(void)`
- **Purpose:** Linear congruential generator (LCG) with separate seed streams
- **Inputs:** None (uses static seed)
- **Outputs/Return:** Next random uint16; updates seed in-place
- **Side effects:** Mutates static seed
- **Calls:** None
- **Notes:** Feedback polynomial 0xb400; XOR if LSB set, else shift; local seed independent for gameplay reproducibility

### long_to_overflow_short_2d() / overflow_short_to_long_2d()
- **Signature:** `void long_to_overflow_short_2d(long_vector2d& LVec, world_point2d& WVec, uint16& flags)` (ref-in, ref-out)
- **Purpose:** Pack/unpack 32-bit coords into short + upper nibbles in flags word
- **Inputs/Outputs:** 32ΓåÆ16-bit packing and vice versa
- **Side effects:** Modifies output refs; sign-extends upper nibbles on unpack
- **Calls:** None
- **Notes:** Stores upper 4 bits of X in flags[15:12], Y in flags[11:8]; kludge for long-distance overflow handling without full 32-bit coordinates

### transform_overflow_point2d()
- **Signature:** `world_point2d *transform_overflow_point2d(world_point2d *point, world_point2d *origin, angle theta, uint16 *flags)`
- **Purpose:** Rotate point, preserving extended coordinates via flags overflow
- **Inputs:** Point, origin, angle, flags word
- **Outputs/Return:** Transformed point with extended coords in flags
- **Side effects:** Mutates point and flags
- **Calls:** `normalize_angle()`, `long_to_overflow_short_2d()`
- **Notes:** Uses 32-bit intermediates internally; result collapsed back to short+flags format

## Control Flow Notes

This file is **initialization and per-frame utility**. `build_trig_tables()` is called at engine startup. Translation, rotation, and distance functions are called frequently during entity updates, collision detection, and rendering to position/test entities and camera. Random functions provide seeded PRNG for deterministic gameplay. The arctangent is used to compute facing angles from velocity/direction vectors. Long-distance overflow functions support map coordinates beyond int16 range.

## External Dependencies

- **Includes:** `<stdlib.h>`, `<math.h>`, `<limits.h>`, `cseries.h` (base types/macros), `world.h` (declarations)
- **Defined elsewhere:** `cosine_table`, `sine_table` (extern globals, initialized here); `TRIG_MAGNITUDE`, `NUMBER_OF_ANGLES`, `NORMALIZE_ANGLE()` macro (world.h); `int16`, `int32`, `uint16` (cstypes.h); `GUESS_HYPOTENUSE()`, `INT16_MAX`, `INT32_MIN` macros
