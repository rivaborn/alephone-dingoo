# Source_Files/RenderMain/AnimatedTextures.h

## File Purpose
Interface for the animated textures subsystem in Aleph One (Marathon engine port). Provides frame-by-frame texture animation during gameplay, allowing surfaces to cycle through animation sequences. Supports XML-based configuration.

## Core Responsibilities
- Update animation state each frame (advance animation timers/counters)
- Translate static texture descriptors to their current animated frame
- Provide XML parser for animated texture configuration

## Key Types / Data Structures
None defined in this file.

| Name | Kind | Purpose |
|------|------|---------|
| `shape_descriptor` | typedef | Encodes texture collection/shape/CLUT indices (from shape_descriptors.h) |
| `XML_ElementParser` | class | XML parsing base class for config (from XML_ElementParser.h) |

## Global / File-Static State
None.

## Key Functions / Methods

### AnimTxtr_Update
- Signature: `void AnimTxtr_Update()`
- Purpose: Advances animation state for all animated textures (increments frame timers, wraps cycles)
- Inputs: None
- Outputs/Return: None
- Side effects: Modifies global/static animation state; called once per frame
- Calls: Not inferable from this file
- Notes: Likely invoked from main game loop during update phase before rendering

### AnimTxtr_Translate
- Signature: `shape_descriptor AnimTxtr_Translate(shape_descriptor Texture)`
- Purpose: Maps a texture descriptor to its current animation frame's descriptor
- Inputs: A shape_descriptor identifying a texture (encodes collection, shape, CLUT)
- Outputs/Return: shape_descriptor pointing to the current frame of the animation
- Side effects: None (read-only lookup)
- Calls: Not inferable from this file
- Notes: Called during rendering to resolve which sprite/texture to display; returns original descriptor if not animated

### AnimatedTextures_GetParser
- Signature: `XML_ElementParser *AnimatedTextures_GetParser()`
- Purpose: Returns the XML element parser for animated texture configuration
- Inputs: None
- Outputs/Return: Pointer to an XML_ElementParser-derived object
- Side effects: None
- Calls: Not inferable from this file
- Notes: Used during engine initialization to parse animation definitions from XML

## Control Flow Notes
- **Update phase**: `AnimTxtr_Update()` called once per frame to advance animation timers
- **Render phase**: `AnimTxtr_Translate()` called for each surface with an animated texture to get the current frame descriptor
- **Init phase**: `AnimatedTextures_GetParser()` called to load animation configuration from XML

## External Dependencies
- `shape_descriptors.h`: Provides `shape_descriptor` typedef and bit-packing macros for encoding texture collection/shape/CLUT information
- `XML_ElementParser.h`: Provides XML parsing infrastructure for configuration files
