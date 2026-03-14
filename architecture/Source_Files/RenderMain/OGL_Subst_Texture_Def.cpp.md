# Source_Files/RenderMain/OGL_Subst_Texture_Def.cpp

## File Purpose
Implements OpenGL substitute texture management for the Aleph One game engine, including loading/unloading textures, caching texture options via hash tables, and XML parsing for texture configuration. Supports per-collection texture organization with fast lookup by CLUT and bitmap ID.

## Core Responsibilities
- Load and unload texture images from collections, computing sprite positioning
- Maintain texture options storage organized by collection with hash-table caching for O(1) lookups
- Parse XML texture configuration elements (`<texture>` and `<txtr_clear>`) into runtime data structures
- Support bulk clearing of texture entries per collection or globally
- Handle texture option replacement and insertion with CLUT-specific vs. generic (ALL_CLUTS) fallbacks

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `OGL_TextureOptions` | struct | Inherits from `OGL_TextureOptionsBase`; holds texture rendering parameters (opacity, blending, image file paths, scaling, positioning) |
| `TextureOptionsEntry` | struct | Container for a texture mapping: CLUT ID, bitmap ID, and associated `OGL_TextureOptions` data |
| `TOList_t` | typedef | `deque<TextureOptionsEntry>` storing texture entries per collection |
| `XML_TO_ClearParser` | class | XML element parser for `<txtr_clear>` elements (clears texture collection) |
| `XML_TextureOptionsParser` | class | XML element parser for `<texture>` elements (defines texture options) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `DefaultTextureOptions` | `OGL_TextureOptions` | static | Default fallback when no specific texture entry matches a lookup |
| `TOList` | `TOList_t[NUMBER_OF_COLLECTIONS]` | static | Array of deques, one per collection, storing texture option entries |
| `TOHash` | `vector<uint16>[NUMBER_OF_COLLECTIONS]` | static | Hash table per collection for O(1) lookup; stores indices into `TOList` with `Specific_CLUT_Flag` bit |
| `TO_ClearParser` | `XML_TO_ClearParser` | static | Singleton XML parser instance for `<txtr_clear>` elements |
| `TextureOptionsParser` | `XML_TextureOptionsParser` | static | Singleton XML parser instance for `<texture>` elements |

## Key Functions / Methods

### OGL_TextureOptions::FindImagePosition
- **Signature:** `void FindImagePosition()`
- **Purpose:** Calculate Right and Bottom sprite corner coordinates based on Left, Top, ImageScale, and actual image dimensions.
- **Inputs:** (implicitly) `Left`, `Top`, `ImageScale`, `NormalImg` width/height
- **Outputs/Return:** Sets `Right` and `Bottom` members
- **Side effects:** Modifies member state
- **Calls:** `NormalImg.GetWidth()`, `NormalImg.GetHeight()`
- **Notes:** Used to adjust sprite texture positions when substitute textures differ in size from originals.

### TODelete
- **Signature:** `void TODelete(short Collection)`
- **Purpose:** Clear all texture option entries for a specified collection.
- **Inputs:** `Collection` ΓÇô collection ID
- **Outputs/Return:** None
- **Side effects:** Clears `TOList[Collection]` and `TOHash[Collection]`
- **Calls:** `deque::clear()`

### OGL_CountTextures
- **Signature:** `int OGL_CountTextures(short Collection)`
- **Purpose:** Return the count of texture entries in a collection.
- **Inputs:** `Collection` ΓÇô collection ID
- **Outputs/Return:** Count of entries in `TOList[Collection]`
- **Side effects:** None
- **Calls:** `deque::size()`

### OGL_LoadTextures
- **Signature:** `void OGL_LoadTextures(short Collection)`
- **Purpose:** Load all texture images in a collection and compute their sprite positioning.
- **Inputs:** `Collection` ΓÇô collection ID
- **Outputs/Return:** None
- **Side effects:** Calls `Load()` on each entry's `OGL_TextureOptions`; calls progress callback
- **Calls:** `OGL_TextureOptions::Load()`, `OGL_TextureOptions::FindImagePosition()`, `OGL_ProgressCallback()`
- **Notes:** Invoked during asset initialization phase.

### OGL_UnloadTextures
- **Signature:** `void OGL_UnloadTextures(short Collection)`
- **Purpose:** Unload all texture images in a collection.
- **Inputs:** `Collection` ΓÇô collection ID
- **Outputs/Return:** None
- **Side effects:** Calls `Unload()` on each entry's `OGL_TextureOptions`

### OGL_GetTextureOptions
- **Signature:** `OGL_TextureOptions *OGL_GetTextureOptions(short Collection, short CLUT, short Bitmap)`
- **Purpose:** Retrieve texture options for a specific CLUT and bitmap, with hash-table caching.
- **Inputs:** `Collection` ΓÇô collection ID; `CLUT` ΓÇô color lookup table (or `ALL_CLUTS`); `Bitmap` ΓÇô bitmap index
- **Outputs/Return:** Pointer to matching `OGL_TextureOptions`, or `&DefaultTextureOptions` if not found
- **Side effects:** Initializes `TOHash[Collection]` on first call; populates hash table on cache miss
- **Calls:** Hash lookup via `TOHashFunc()`, linear deque search on cache miss
- **Notes:** Hash uses `Specific_CLUT_Flag` bit to track whether match was CLUT-specific or generic. Prefers exact CLUT match over `ALL_CLUTS` entry (specific entries stored at front, generic at back).

### XML_TO_ClearParser::Start / HandleAttribute / AttributesDone
- **Purpose:** Parse `<txtr_clear>` XML elements to clear texture collections.
- **Key logic:** If `coll` attribute present, clear that collection; otherwise clear all.
- **Calls:** `TODelete()`, `TODelete_All()`, `ReadBoundedInt16Value()`

### XML_TextureOptionsParser::Start / HandleAttribute / AttributesDone / ResetValues
- **Purpose:** Parse `<texture>` XML elements, populate `OGL_TextureOptions` data, and insert/replace into `TOList`.
- **Key logic:** Parses collection, CLUT, bitmap, opacity, blending, image paths, and positioning attributes. Replaces existing entry or inserts new. Specific-CLUT entries inserted at front for priority.
- **Calls:** `ReadBoundedInt16Value()`, `ReadFloatValue()`, `ReadBooleanValueAsBool()`, `ReadString()`, `ReadInt16Value()`, `TODelete_All()`

## Control Flow Notes
**Initialization phase:** XML parsers are instantiated statically and registered with the XML parsing system.  
**Asset loading:** `OGL_LoadTextures()` is called per collection to load images and compute sprite positions.  
**Rendering setup:** `OGL_GetTextureOptions()` is called at texture binding time to retrieve options, using hash-table cache for performance.  
**Cleanup:** `OGL_UnloadTextures()` called to release texture images; `TODelete()` called when reloading configurations.

## External Dependencies
- **cseries.h** ΓÇô utility macros, types, and string parsing helpers
- **OGL_Texture_Def.h** ΓÇô `OGL_TextureOptionsBase` struct definition
- **XML_ElementParser.h** ΓÇô `XML_ElementParser` base class
- **deque, vector** ΓÇô STL containers
- **Defined elsewhere:** `OGL_ProgressCallback()`, `StringsEqual()`, `ReadBoundedInt16Value()`, `ReadFloatValue()`, `ReadBooleanValueAsBool()`, `ReadInt16Value()`, `OGL_TextureOptions::Load()`, `OGL_TextureOptions::Unload()`, `OGL_TextureOptionsBase` definition, `NUMBER_OF_COLLECTIONS`, `MAXIMUM_SHAPES_PER_COLLECTION`, XML parsing infrastructure
