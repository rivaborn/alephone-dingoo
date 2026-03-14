# Source_Files/RenderMain/OGL_Texture_Def.h

## File Purpose

Header file defining base OpenGL texture structures and enumerations for the Aleph One game engine. Provides shared configuration and metadata for wall/sprite texture substitutions and model skins in OpenGL rendering. Authored by Loren Petrich, May 2003.

## Core Responsibilities

- Define bitmap set indices (infravision, silhouette) and enumeration constants for OpenGL texture management
- Enumerate opacity types (Crisp, Flat, Avg, Max) controlling alpha channel interpretation
- Enumerate blend types (Crossfade, Add, with premultiply variants) for texture compositing
- Provide `OGL_TextureOptionsBase` struct for shared texture configuration across wall/sprite and skin textures
- Declare Load/Unload methods and size querying for texture resource lifecycle

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `OGL_TextureOptionsBase` | struct | Base configuration container for OpenGL textures, holding opacity/blend settings, file paths, image descriptors, and premultiplication flags |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `INFRAVISION_BITMAP_SET` | enum | global | Bitmap set index for infravision texture variants |
| `SILHOUETTE_BITMAP_SET` | enum | global | Bitmap set index for silhouette texture variants |
| `NUMBER_OF_OPENGL_BITMAP_SETS` | enum | global | Total count of bitmap set types |
| `ALL_CLUTS` | const int | global | Sentinel value (-1) indicating "all color tables" |
| Opacity type constants | enum | global | `OGL_OpacType_Crisp`, `OGL_OpacType_Flat`, `OGL_OpacType_Avg`, `OGL_OpacType_Max` |
| Blend type constants | enum | global | `OGL_BlendType_Crossfade`, `OGL_BlendType_Add`, premultiply variants |

## Key Functions / Methods

### OGL_TextureOptionsBase::Load
- **Purpose:** Load texture data from configured file paths into memory
- **Inputs:** None (uses member file path vectors)
- **Outputs/Return:** void
- **Side effects:** Allocates memory, loads image files, populates `NormalImg`/`GlowImg` descriptors

### OGL_TextureOptionsBase::Unload
- **Purpose:** Release texture data from memory
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** Deallocates image buffers

### OGL_TextureOptionsBase::GetMaxSize
- **Purpose:** Query maximum texture dimension for quality tier selection
- **Inputs:** None
- **Outputs/Return:** int (max dimension or memory limit)
- **Notes:** Virtual method; subclasses override for type-specific limits

### OGL_TextureOptionsBase (constructor)
- **Signature:** `OGL_TextureOptionsBase()`
- **Purpose:** Initialize texture options to defaults
- **Notes:** Sets opacity to Crisp, scale/shift to default, blending to Crossfade, clears file vectors and image descriptors

## Control Flow Notes

This file defines initialization/configuration structures for the rendering pipeline. Texture options are populated during asset loading (likely from config files or collections), then passed to rendering subsystems. Load/Unload bracket texture resource lifetime relative to scene/frame rendering.

## External Dependencies

- **Includes:** `<vector>` (STL containers), `shape_descriptors.h` (collection/shape identifiers), `ImageLoader.h` (ImageDescriptor class)
- **Conditional:** Entire file guarded by `#ifdef HAVE_OPENGL`
