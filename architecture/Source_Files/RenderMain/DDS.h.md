# Source_Files/RenderMain/DDS.h

## File Purpose
Defines the DirectDraw Surface (DDS) file format specification for texture loading. Provides structure and constant definitions matching the DirectX 9 DDS file format, enabling parsing and interpretation of DDS texture files in the rendering pipeline.

## Core Responsibilities
- Define DDS surface descriptor flags (DDSD_* constants) to indicate which fields are present
- Define pixel format flags (DDPF_* constants) for color/compression modes
- Define surface capability flags (DDSCAPS_*, DDSCAPS2_* constants) for texture properties
- Provide DDSURFACEDESC2 structure matching the official DDS binary format specification
- Guard against double-inclusion with __DDRAW_INCLUDED__ check

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| DDSURFACEDESC2 | struct | Main DDS surface descriptor containing dimensions, pitch, mipmaps, pixel format, and surface capabilities; mirrors DirectX 9 binary layout |

## Global / File-Static State
None.

## Key Functions / Methods
NoneΓÇöthis is a data structure and constant definition header.

## Control Flow Notes
Not inferable from this file. Used during asset loading when parsing DDS texture files; structure layout must exactly match the DirectX 9 DDS binary format for correct file parsing.

## External Dependencies
- **cstypes.h**: Provides `uint32` type definition
- Conditional guard: `__DDRAW_INCLUDED__` (skips redefinition if DirectX headers already included)
