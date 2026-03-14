# Source_Files/CSeries/gdspec.h

## File Purpose
Header file defining graphics device specification structures and functions for the Aleph One game engine. Provides abstractions for querying, matching, and configuring graphics device capabilities (resolution, bit depth, slot).

## Core Responsibilities
- Define the `GDSpec` structure to encapsulate graphics device parameters
- Declare functions for finding and matching compatible graphics devices
- Provide graphics device depth/bit-depth configuration and validation
- Support graphics device comparison and querying (slot, capabilities)
- Present a device selection dialog to the user

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| GDSpec | struct | Holds graphics device specification: slot, flags, bit depth, width, height |
| GDSpecPtr | typedef | Pointer to GDSpec; used throughout for parameter passing |

## Global / File-Static State
None.

## Key Functions / Methods

### BestDevice
- Signature: `GDHandle BestDevice(GDSpecPtr spec)`
- Purpose: Find the best-matching graphics device for the given spec
- Inputs: Pointer to GDSpec with desired configuration
- Outputs/Return: GDHandle (graphics device handle)
- Notes: Selection criteria not inferable from header

### MatchGDSpec
- Signature: `GDHandle MatchGDSpec(GDSpecPtr spec)`
- Purpose: Find an exact or closest match to the given graphics device spec
- Inputs: Pointer to GDSpec with target parameters
- Outputs/Return: GDHandle to matched device

### SetDepthGDSpec
- Signature: `void SetDepthGDSpec(GDSpecPtr spec)`
- Purpose: Update the spec with the current or appropriate bit-depth for a graphics device
- Inputs: Pointer to GDSpec to modify
- Side effects: Modifies the `bit_depth` field of spec

### BuildGDSpec
- Signature: `void BuildGDSpec(GDSpecPtr spec, GDHandle dev)`
- Purpose: Populate a GDSpec structure from an existing graphics device handle
- Inputs: Pointer to GDSpec to populate, graphics device handle
- Side effects: Fills spec with device's current configuration (slot, dimensions, depth)

### HasDepthGDSpec / EqualGDSpec
- Signature: `bool HasDepthGDSpec(GDSpecPtr spec)`, `bool EqualGDSpec(GDSpecPtr spec1, GDSpecPtr spec2)`
- Purpose: Query depth support and compare two specs for equality
- Inputs: One or two GDSpec pointers
- Outputs/Return: Boolean result

### GetSlotFromGDevice
- Signature: `short GetSlotFromGDevice(GDHandle dev)`
- Purpose: Extract the slot number from a graphics device handle
- Inputs: GDHandle
- Outputs/Return: Short integer (slot ID)

### display_device_dialog
- Signature: `void display_device_dialog(GDSpecPtr spec)`
- Purpose: Present a user dialog for selecting/configuring a graphics device
- Inputs: Pointer to GDSpec (likely for initial state or output)
- Side effects: User interaction, possible modification of spec

## Control Flow Notes
This is a header file; no control flow visible. Functions are platform/device configuration utilities, likely called during engine initialization (graphics setup) or user settings changes.

## External Dependencies
- `GDHandle` and `GDDevice` types: defined elsewhere (likely macOS Carbon API or engine-internal graphics system)
- Aleph One game engine copyright and GPL v2 license
