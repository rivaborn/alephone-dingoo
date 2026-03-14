# Source_Files/RenderOther/ViewControl.h

## File Purpose
View controller module for the Marathon/Aleph One game engine. Controls viewing parameters including field-of-view, landscape rendering options, teleport effects, and on-screen display settings. Provides XML-based configuration support for customizing these visual parameters.

## Core Responsibilities
- Manage field-of-view (FOV) states: normal, extravision, and tunnel vision modes
- Provide smooth FOV adjustments toward target values
- Control visual effects for teleportation (fold effect, static effect, interlevel effects)
- Query and configure landscape rendering parameters
- Provide on-screen display font access
- Manage XML-based configuration for all view settings
- Check overhead map availability status

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| LandscapeOptions | struct | Configuration for landscape rendering including horizontal/vertical scaling, OpenGL aspect ratio, texture repeat behavior, and azimuth rotation |

## Global / File-Static State
None.

## Key Functions / Methods

### View_AdjustFOV
- Signature: `bool View_AdjustFOV(float& FOV, float FOV_Target)`
- Purpose: Smoothly transition field-of-view toward a target value
- Inputs: `FOV` (current FOV, passed by reference); `FOV_Target` (desired FOV value)
- Outputs/Return: `bool` ΓÇô true if FOV was changed, false if already at target
- Side effects: Modifies the FOV reference parameter
- Calls: Not visible in this file
- Notes: Enables gradual FOV transitions; caller provides storage for FOV state

### View_GetLandscapeOptions
- Signature: `LandscapeOptions *View_GetLandscapeOptions(shape_descriptor Desc)`
- Purpose: Retrieve landscape rendering configuration for a given shape descriptor
- Inputs: `Desc` (shape descriptor combining collection, clut, and shape IDs)
- Outputs/Return: Pointer to LandscapeOptions struct (null if not found)
- Side effects: None visible
- Calls: Not visible in this file
- Notes: Landscape configuration is per-shape-descriptor; used during landscape rendering

### View_GetParser
- Signature: `XML_ElementParser *View_GetParser()`
- Purpose: Obtain XML parser for view configuration elements
- Inputs: None
- Outputs/Return: Pointer to XML_ElementParser
- Side effects: None visible
- Calls: Not visible in this file
- Notes: Used during XML configuration loading to apply view settings

### Landscapes_GetParser
- Signature: `XML_ElementParser *Landscapes_GetParser()`
- Purpose: Obtain XML parser for landscape configuration elements
- Inputs: None
- Outputs/Return: Pointer to XML_ElementParser
- Side effects: None visible
- Calls: Not visible in this file
- Notes: Separate parser for landscape-specific XML settings

## Control Flow Notes
This header declares query and configuration functions. Typical integration:
- FOV and effect flag functions are called during frame updates to query current settings
- `View_AdjustFOV()` is likely called each frame to smoothly animate FOV changes
- Landscape options are queried during landscape rendering setup
- XML parsers are invoked during initialization/configuration loading
- Teleport effect flags (`View_DoFoldEffect()`, `View_DoStaticEffect()`, etc.) are checked when teleportation events occur

## External Dependencies
**Includes:**
- `world.h` ΓÇô World coordinates, angle type definition
- `FontHandler.h` ΓÇô FontSpecifier class
- `shape_descriptors.h` ΓÇô shape_descriptor typedef and collection/clut macros
- `XML_ElementParser.h` ΓÇô XML parsing framework

**External symbols used but not defined here:**
- `FontSpecifier` (class)
- `shape_descriptor` (typedef'd as uint16)
- `angle` (typedef'd as int16)
- `XML_ElementParser` (class)
