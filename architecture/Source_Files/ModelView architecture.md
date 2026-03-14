# Subsystem Overview

## Purpose
The ModelView subsystem loads, stores, and renders 3D model geometry with skeletal animation for the Aleph One engine. It provides multiple loaders for different 3D model formats (Wavefront OBJ, QuickDraw 3D/Quesa, 3D Studio Max binary, Dim3 XML) that populate a unified Model3D structure, with rendering support for multi-pass shading, depth-sorted polygon drawing, and bone-based animation.

## Key Files
| File | Role |
|------|------|
| **Model3D.h** | Core 3D asset container: defines structures for vertex data, skeletal bones, animation frames/sequences, transformation matrices, and bounding box management |
| **Model3D.cpp** | Implements skeletal animation (bone matrices, dual-bone vertex blending), vertex position computation (neutral/frame/sequence modes), normal processing, and bounding box calculation |
| **ModelRenderer.h** | Defines renderer class for multi-pass shading with optional Z-buffer; manages depth-sorting of triangles by centroid when Z-buffer unavailable |
| **ModelRenderer.cpp** | Implements multi-pass rendering with shader callbacks for texturing and per-vertex lighting; depth-sorts polygons and manages OpenGL state (vertex arrays, textures, colors) |
| **Dim3_Loader.h** | Public interface for loading Dim3 XML model format with multi-pass control and debug output configuration |
| **Dim3_Loader.cpp** | Parses Dim3 XML via XML_Configure framework; loads vertices (positions, normals, dual-bone weights), bone hierarchy, animation frames/sequences; converts Dim3 conventions to engine units |
| **WavefrontLoader.h** | Public interface for loading Wavefront OBJ (.obj) format models |
| **WavefrontLoader.cpp** | Line-by-line OBJ parser with vertex deduplication, index remapping (1-based to 0-based), negative index support, and polygon triangulation |
| **QD3D_Loader.h** | Public interface for loading QuickDraw 3D / Quesa format (.3dmf) models with tesselation parameter configuration |
| **QD3D_Loader.cpp** | Quesa library initialization and model loading; custom triangulator renderer; vertex deduplication by position and attribute thresholds |
| **StudioLoader.h** | Public interface for loading 3D Studio Max (.3ds) binary format models |
| **StudioLoader.cpp** | Binary chunk-parser for .3DS files; hierarchical chunk traversal (MASTERΓåÆEDITORΓåÆOBJECTΓåÆTRIMESH); extracts vertices, texture coordinates, face indices with little-endian byte order |

## Core Responsibilities
- **Model loading**: Multiple format parsers (OBJ, QD3D/Quesa, 3DS binary, Dim3 XML) that populate a unified Model3D structure
- **3D geometry storage**: Vertex positions, texture coordinates, normals, vertex colors in OpenGL-friendly contiguous arrays; support for vertex deduplication
- **Skeletal animation**: Bone hierarchy management with parent-child relationships; per-bone transformation matrices; dual-bone weighted vertex blending
- **Animation sequences**: Frame definitions (sets of bone poses) and animation sequences (ordered frame lists with optional crossfading)
- **Vertex position computation**: Three modesΓÇöneutral static pose, single animation frame, or sequence animation with interpolation
- **Normal processing**: Calculation, normalization, reversal, and optional smoothing with hard edge splitting
- **Rendering with multiple shading modes**: Multi-pass rendering, per-triangle or per-vertex shader callbacks for lighting and texturing, Z-buffer optimized rendering
- **Depth-sorted rendering**: Triangle centroid-based depth sorting when Z-buffer unavailable; maintains sorted polygon order across passes

## Key Interfaces & Data Flow

**Exposures to other subsystems:**
- `Model3D` class: unified 3D asset container with methods to compute vertex positions at animation frames/sequences
- `ModelRenderer` class: multi-pass rendering interface accepting shader callbacks for texture/lighting configuration
- Loader functions: `LoadModel_Dim3()`, `LoadModel_Wavefront()`, `LoadModel_QD3D()`, `LoadModel_3DS()` ΓÇö all populate `Model3D` from `FileSpecifier`

**Consumes from other subsystems:**
- **Model3D.h/cpp**: Uses global `BuildTrigTables()` (from `world.h`), Marathon trig lookups (`cosine_table`, `sine_table`, `TRIG_MAGNITUDE`), angle utilities (`NORMALIZE_ANGLE`, `FULL_CIRCLE`)
- **ModelRenderer**: Uses OpenGL state setup (`glEnableClientState`, `glVertexPointer`, `glTexCoordPointer`, `glColorPointer`, `glDrawElements`)
- **Dim3_Loader**: Uses `XML_Configure` framework and `XML_ElementParser` base; reads files via `FileHandler` (`FileSpecifier`, `OpenedFile`)
- **QD3D_Loader**: Uses Quesa library (Q3Initialize, Q3File_*, Q3Renderer_*, Q3AttributeSet_*)
- **StudioLoader / WavefrontLoader**: Use `FileHandler` abstractions and utility macros from `cseries.h`

## Runtime Role
The ModelView subsystem is **asset-loading and rendering-time**: loaders execute during model file loading (initialization or streaming), populate Model3D structures that persist in memory. ModelRenderer is invoked during frame rendering to draw 3D models with animation frame updates and multi-pass shading. Bone transformation matrices and vertex positions are recalculated when animation state changes or per-frame during playback.

## Notable Implementation Details
- **Dim3_Loader**: Uses XML_Configure base class for hierarchical element parsing; remaps vertex bone indices to canonical order via tree traversal of bone hierarchy
- **WavefrontLoader**: Handles 1-based OBJ vertex indexing with support for negative (from-end) indices; triangulates arbitrary polygons into triangle fans
- **QD3D_Loader**: Lazy initialization of Quesa library; custom fake renderer (triangulator) to extract geometry from curved surface objects; deduplicates vertices within position/attribute thresholds
- **StudioLoader**: Little-endian binary chunk format; hierarchical chunk structure validation (checks MASTER chunk identifier)
- **ModelRenderer**: Conditional depth-sorting path when Z-buffer unavailable; separable shader optimization when Z-buffer present; maintains persistent scratch buffers (centroid depths, sorted indices, lighting colors) to minimize re-allocation
