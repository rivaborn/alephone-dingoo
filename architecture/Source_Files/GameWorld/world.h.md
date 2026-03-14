# Source_Files/GameWorld/world.h

## File Purpose
Header defining the fundamental coordinate system, geometric types, and transformation utilities for the game world. Establishes fixed-point arithmetic conventions, angle representation (512 angles per circle), and 2D/3D point/vector structures used throughout the engine.

## Core Responsibilities
- Define fixed-point world coordinate types (`world_distance`, `angle`) and their conversion macros
- Declare geometric structures: world points/vectors (2D, 3D), fixed-point variants, and long-integer overflow variants
- Provide angle normalization and discretized facing-direction macros (4/5/8-way)
- Declare geometric transformation functions (rotate, translate, transform) for 2D and 3D
- Declare trigonometric table access and angle calculation (`arctangent`)
- Provide distance/hypotenuse calculation utilities
- Declare random number generation and seed management
- Declare overflow-safe 2D transformation variants for long-distance calculations

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| world_point2d, world_point3d | struct | 2D/3D position using fixed-point `world_distance` (10 fractional bits) |
| world_vector2d, world_vector3d | struct | 2D/3D displacement vector in world coordinates |
| fixed_point3d, fixed_vector3d | struct | 3D point/vector using `_fixed` arithmetic |
| long_point2d/3d, long_vector2d/3d | struct | 32-bit integer variants for intermediate calculations without overflow |
| world_location3d | struct | Composite: 3D position, polygon index, yaw/pitch angles, velocity vector |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| cosine_table | short\* | extern | Precomputed cosine values indexed by normalized angle |
| sine_table | short\* | extern | Precomputed sine values indexed by normalized angle |

## Key Functions / Methods

### build_trig_tables
- Signature: `void build_trig_tables(void);`
- Purpose: Initialize sine and cosine lookup tables
- Side effects: Populates global `cosine_table` and `sine_table`
- Notes: Must be called during engine initialization

### rotate_point2d / rotate_point3d
- Signature: `world_point2d *rotate_point2d(world_point2d *point, world_point2d *origin, angle theta);` (similar for 3D)
- Purpose: Rotate a point around an origin by angle theta
- Inputs: Point to rotate, rotation center, rotation angle
- Outputs: Modified point (in-place return)
- Calls: Uses `cosine_table`, `sine_table`

### translate_point2d / translate_point3d
- Purpose: Displace a point by distance along angle theta
- Inputs: Point, distance magnitude, direction angle (theta for 2D; theta, phi for 3D)
- Outputs: Modified point

### transform_point2d / transform_point3d
- Purpose: Combined rotation around origin followed by translation
- Inputs: Point, origin, angle(s)
- Outputs: Transformed point

### arctangent
- Signature: `angle arctangent(int32 x, int32 y);`
- Purpose: Calculate angle from Cartesian coordinates
- Inputs: x, y displacement (32-bit for long-distance friendliness)
- Outputs: Angle in `[0, NUMBER_OF_ANGLES)` (512 steps per circle)
- Notes: Accepts larger integer range than `world_distance` to avoid intermediate overflow

### distance2d / distance3d
- Purpose: Calculate Euclidean distance between two points
- Signature: `world_distance distance2d(world_point2d *p0, world_point2d *p1);`
- Notes: `distance2d` calls `isqrt()` for square root

### guess_distance2d
- Purpose: Fast approximate distance (Chebyshev variant: `max(x,y) + 0.5*min(x,y)`)
- Uses macro `GUESS_HYPOTENUSE()`

### set_random_seed / get_random_seed / global_random / local_random
- Signature: `void set_random_seed(uint16 seed);` `uint16 get_random_seed(void);` `uint16 global_random(void);` `uint16 local_random(void);`
- Purpose: Global and local random number generation with seed control

### transform_overflow_point2d
- Signature: `world_point2d *transform_overflow_point2d(world_point2d *point, world_point2d *origin, angle theta, uint16 *flags);`
- Purpose: Transform that stores overflow bits in a separate flags word to handle coordinates beyond short range
- Inputs: Point, origin, angle, flags word (in/out)
- Notes: Part of long-distance calculation kludge

**Inline helpers:** `long_to_short_2d()`, `short_to_long_2d()` ΓÇö simple cast conversions between 32-bit and 16-bit vector formats.

## Control Flow Notes
This file establishes coordinate system conventions used globally. Initialization flow: `build_trig_tables()` called once at engine startup. Transformation functions are called during entity updates (movement, rotation) and input processing (aiming, distance checks). The overflow variants (`long_*`, `overflow_*`) are used for high-precision calculations over large world distances to avoid integer overflow in intermediate results.

## External Dependencies
- `_fixed`, `_fixed` arithmetic (defined elsewhere; used for fixed-point types)
- `isqrt()` ΓÇö integer square root (likely in `world.c`)
- Trigonometric tables declared but populated by `build_trig_tables()` in `world.c`
- `FIXED_FRACTIONAL_BITS` macro (defined elsewhere, used in coordinate conversions)
