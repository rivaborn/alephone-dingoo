# Source_Files/RenderMain/OGL_Faders.cpp

## File Purpose
Implements OpenGL-based screen fading effects for the Aleph One game engine. Manages a queue of fader effects (tints, static, negation, dodge, burn) and applies them to the rendered framebuffer using various OpenGL blending modes and color operations.

## Core Responsibilities
- Manage a queue of pending fader effects and their parameters
- Implement six distinct fader types with different visual algorithms
- Handle color manipulation utilities (alpha multiplication, complement)
- Coordinate OpenGL state management for blending and color operations
- Support both sRGB-corrected and standard color rendering
- Provide configuration-driven mode selection (flat static vs. logic-op randomization)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| OGL_Fader | struct | Holds fader type and RGBA color; defined in OGL_Faders.h |
| GM_Random | struct | Random number generator for static effect colors; from Random.h |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| FlatStaticRandom | GM_Random | static | Generates pseudo-random colors for flat-static fading effect |
| UseFlatStatic | bool | static | Flag: use flat static or XOR logic-op for randomize fader |
| FlatStaticColor[4] | uint16[4] | static | RGBA color buffer for flat-static effect output |
| FaderQueue[2] | OGL_Fader[] | static | Queue of pending faders (liquid + other categories) |

## Key Functions / Methods

### OGL_FaderActive
- Signature: `bool OGL_FaderActive()`
- Purpose: Determine whether OpenGL fader rendering is enabled in the current configuration
- Inputs: None
- Outputs/Return: `true` if OpenGL is active AND the fader flag is set
- Side effects: None
- Calls: `OGL_IsActive()`, `Get_OGL_ConfigureData()`, `TEST_FLAG()`
- Notes: Guard function; `OGL_DoFades()` returns false immediately if this returns false

### GetOGL_FaderQueueEntry
- Signature: `OGL_Fader *GetOGL_FaderQueueEntry(int Index)`
- Purpose: Accessor for a specific fader in the queue
- Inputs: Queue index (0 or 1)
- Outputs/Return: Pointer to the OGL_Fader struct at that index
- Side effects: Asserts if index is out of bounds
- Calls: None
- Notes: Simple bounds-checked array accessor

### MultAlpha
- Signature: `inline void MultAlpha(GLfloat *InColor, GLfloat *OutColor)`
- Purpose: Multiply RGB channels by alpha; used to pre-multiply alpha for blending
- Inputs: 4-element RGBA color array
- Outputs/Return: Separate output array with RGB scaled by alpha, alpha unchanged
- Side effects: None
- Calls: None
- Notes: Does not modify input; essential for correct blending in several fader types

### ComplementColor
- Signature: `inline void ComplementColor(GLfloat *InColor, GLfloat *OutColor)`
- Purpose: Invert RGB channels (1 - R, 1 - G, 1 - B); used for dodge and burn effects
- Inputs: 4-element RGBA color array
- Outputs/Return: Separate output array with inverted RGB, alpha unchanged
- Side effects: None
- Calls: None

### OGL_DoFades
- Signature: `bool OGL_DoFades(float Left, float Top, float Right, float Bottom)`
- Purpose: Render all queued fader effects as screen-space overlays within the given rectangle
- Inputs: Screen bounds in normalized/screen coordinates
- Outputs/Return: `true` if any faders were processed; `false` if faders inactive
- Side effects: 
  - Modifies OpenGL state: blend modes, color logic ops, texture state
  - Draws one or two GL_POLYGON quads per active fader
  - Temporarily disables/restores GL_ALPHA_TEST, GL_TEXTURE_2D, GL_BLEND, GL_COLOR_LOGIC_OP, GL_FRAMEBUFFER_SRGB_EXT
- Calls: 
  - `OGL_FaderActive()`, `glVertexPointer()`, `glDisable()`, `glEnable()`
  - `glDrawArrays()`, `glBlendFunc()`, `glLogicOp()`, `glColor3fv()`, `glColor4fv()`
  - `SglColor4fva()`, `SglColor4usv()`, `MultAlpha()`, `ComplementColor()`
  - `FlatStaticRandom.KISS()`, `FlatStaticRandom.LFIB4()`
  - `Get_OGL_ConfigureData()`, `TEST_FLAG()`
- Notes: 
  - Early return if `OGL_FaderActive()` is false
  - Builds a full-screen quad (4 vertices) for rendering
  - Switches blend function for each fader type; always restores to `GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA` after
  - **_tint_fader_type**: Simple overlay with fader color
  - **_randomize_fader_type**: Either flat static mode (random colors + alpha blending) or XOR logic-op (flips low bits per opacity)
  - **_negate_fader_type**: Approximation using `GL_ONE_MINUS_DST_COLOR` blend
  - **_dodge_fader_type**: Double-pass with inverted color and `GL_DST_COLOR` blending
  - **_burn_fader_type**: Two-pass: original color with `GL_DST_COLOR, GL_ONE`, then complement with `GL_SRC_ALPHA, GL_ONE`
  - **_soft_tint_fader_type**: Illumination effect; applies sRGB gamma correction to color before blending with `GL_DST_COLOR, GL_ONE_MINUS_SRC_ALPHA`

## Control Flow Notes
Called during each frame's rendering after the main 3D scene is complete but before buffer swap. Iterates through the static fader queue (`FaderQueue[0..1]`) and applies effects in order. OpenGL state is carefully managed to avoid affecting subsequent 2D/UI rendering. The function respects the global `Using_sRGB` flag for colorspace-aware blending.

## External Dependencies
- **fades.h**: Fader type enums (`NONE`, `_tint_fader_type`, `_randomize_fader_type`, etc.)
- **Random.h**: `GM_Random` class (KISS/LFIB4 random generators)
- **OGL_Setup.h**: `OGL_IsActive()`, `Get_OGL_ConfigureData()`, `SglColor*()` wrappers, `Using_sRGB` flag, `sRGB_frob()`, `OGL_Flag_*` constants
- **OGL_Render.h**: (included via OGL_Setup.h or indirectly)
- **OpenGL headers**: Platform-conditional (`<OpenGL/gl.h>`, `<GL/gl.h>`, `<gl.h>`)
- **cseries.h**: Base macros and types (`TEST_FLAG`, `assert`)
