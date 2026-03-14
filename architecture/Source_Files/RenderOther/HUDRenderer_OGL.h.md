# Source_Files/RenderOther/HUDRenderer_OGL.h

## File Purpose
OpenGL-specific HUD renderer implementation. Declares `HUD_OGL_Class`, which inherits from the base `HUD_Class` and provides concrete OpenGL rendering methods for the game's heads-up display (motion sensor, weapons panel, inventory, etc.).

## Core Responsibilities
- Override virtual rendering methods with OpenGL implementations
- Handle motion sensor updates and rendering
- Draw and manage entity blips (enemy/ally markers) on the HUD
- Provide texture and shape drawing via OpenGL
- Support dynamic clipping planes for bounded HUD elements
- Implement text and rectangle rendering (fill/frame)

## Key Types / Data Structures
None (class-only; inherits from `HUD_Class`).

## Global / File-Static State
None.

## Key Functions / Methods

### HUD_OGL_Class (constructor/destructor)
- Signature: `HUD_OGL_Class() {}`, `~HUD_OGL_Class() {}`
- Purpose: Trivial initialization/cleanup for OpenGL HUD renderer
- Notes: Empty bodies; all setup likely deferred to base class or implementation file

### update_motion_sensor
- Signature: `void update_motion_sensor(short time_elapsed)`
- Purpose: Update motion sensor state (e.g., entity positions, scan animation)
- Inputs: `time_elapsed` ΓÇö elapsed time since last update
- Outputs/Return: None
- Side effects: Likely modifies internal motion sensor state
- Notes: Pure virtual override; implementation in .cpp file

### render_motion_sensor
- Signature: `void render_motion_sensor(short time_elapsed)`
- Purpose: Render motion sensor display using OpenGL
- Inputs: `time_elapsed` ΓÇö elapsed time since last frame
- Outputs/Return: None
- Side effects: OpenGL draw calls
- Notes: Pure virtual override; implementation in .cpp file

### draw_entity_blip
- Signature: `void draw_entity_blip(point2d *location, shape_descriptor shape)`
- Purpose: Draw a single entity blip (enemy/ally marker) at given location
- Inputs: `location` ΓÇö 2D screen position; `shape` ΓÇö shape descriptor for blip appearance
- Outputs/Return: None
- Side effects: OpenGL draw calls
- Notes: Counterpart to `erase_entity_blip`; pure virtual override

### erase_entity_blip
- Signature: `void erase_entity_blip(point2d *location, shape_descriptor shape) {}`
- Purpose: Erase entity blip (likely clearing or masking)
- Notes: Empty stub; OpenGL may use other clearing mechanisms

### DrawShape / DrawShapeAtXY / DrawText / FillRect / FrameRect / DrawTexture / SetClipPlane / DisableClipPlane
- All pure virtual overrides of base class methods
- Provide OpenGL-based implementations for fundamental HUD drawing primitives
- Signatures match base class; implementations delegated to .cpp file

## Control Flow Notes
This file is header-only declaration; actual control flow lives in the implementation (.cpp). It acts as the **OpenGL rendering backend** for the HUD subsystem. Called from the main HUD update/render loop (`update_everything` in `HUD_Class`) once per frame. Conditioned by `#ifdef HAVE_OPENGL` at compile time.

## External Dependencies
- `HUDRenderer.h` ΓÇö base class (`HUD_Class`) and shared HUD constants/structures
- `config.h` ΓÇö conditional compilation (`HAVE_OPENGL` guard)
