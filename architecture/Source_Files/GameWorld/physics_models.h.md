# Source_Files/GameWorld/physics_models.h

## File Purpose
Defines character physics models (walking, running) and parameters controlling velocity, acceleration, rotation, and collision dimensions. Provides serialization interfaces for physics constants data.

## Core Responsibilities
- Define physics model enumeration (_model_game_walking, _model_game_running)
- Declare physics_constants structure with all movement/collision parameters
- Store original hardcoded physics constants for both models
- Provide pack/unpack functions for physics constant serialization
- Declare initialization function for physics model system

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| physics_constants | struct | Contains all physics parameters: velocity limits, accelerations, angular motion, gravity, and collision/camera dimensions |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| physics_models | physics_constants[NUMBER_OF_PHYSICS_MODELS] | static | Mutable physics model data, indexed by model enum; can be modified at runtime |
| original_physics_models | physics_constants[NUMBER_OF_PHYSICS_MODELS] | const | Immutable hardcoded defaults for walking and running models |

## Key Functions / Methods

### unpack_physics_constants
- Signature: `uint8 *unpack_physics_constants(uint8 *Stream, physics_constants *Objects, size_t Count)`
- Purpose: Deserialize physics constants from a byte stream (level files, network data)
- Inputs: Stream pointer, Objects array to populate, Count of objects
- Outputs/Return: Updated stream pointer (enables chaining)
- Side effects: Modifies Objects array; advances stream
- Notes: Comment indicates added for "1-2-3 Converter" tool compatibility; implementation elsewhere

### pack_physics_constants
- Signature: `uint8 *pack_physics_constants(uint8 *Stream, physics_constants *Objects, size_t Count)`
- Purpose: Serialize physics constants to a byte stream
- Inputs: Stream pointer, Objects array, Count of objects
- Outputs/Return: Updated stream pointer
- Side effects: Writes to Stream; advances pointer
- Notes: Inverse of unpack; used for persistence/transmission

### init_physics_constants
- Signature: `void init_physics_constants()`
- Purpose: Initialize physics model system (likely copies original_physics_models into mutable physics_models)
- Side effects: Populates global physics_models
- Notes: Called during engine startup

## Control Flow Notes
Initialized early in engine startup via init_physics_constants(). Physics_models indexed and accessed each frame during character movement simulation. Pack/unpack called during level load/save or network synchronization.

## External Dependencies
- `_fixed` type (fixed-point arithmeticΓÇödefined elsewhere)
- `FIXED_ONE`, `QUARTER_CIRCLE` constants (defined elsewhere)
- Standard C types: `uint8`, `size_t`
