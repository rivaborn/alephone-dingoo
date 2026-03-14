# Source_Files/RenderMain/render.h

## File Purpose

Core rendering subsystem interface. Defines the view configuration structure (`view_data`), rendering constants/flags for visibility culling, and declares the primary render function and helper routines for effects and UI rendering.

## Core Responsibilities

- Define `view_data` structure containing all camera/viewport parameters (FOV, screen dimensions, origin, orientation, effects state)
- Manage render flags (`_polygon_is_visible_bit`, `_endpoint_is_visible_bit`, etc.) for visibility culling across map geometry
- Declare main rendering entry point (`render_view`) and initialization
- Support render effects (fold-in, fold-out, explosions)
- Provide texture transfer mode instantiation for special rendering modes
- Manage overhead map and computer interface rendering
- Control shading modes (normal, infravision)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `point2d` | struct | 2D screen coordinates (x, y) |
| `definition_header` | struct | Clip bounds header for rendering definitions |
| `view_data` | struct | Complete view configuration: FOV, screen dims, camera pos/orientation, effects, media state, tunnel vision flag |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `RenderFlagList` | `vector<uint16>` | global | Per-geometry visibility/transform flags; accessed via `render_flags` macro |

## Key Functions / Methods

### allocate_render_memory
- Signature: `void allocate_render_memory(void)`
- Purpose: Initialize/allocate rendering subsystem memory
- Inputs: None
- Outputs/Return: None
- Side effects: Allocates RenderFlagList and other render buffers
- Calls: Not visible in header
- Notes: Must be called before any rendering

### initialize_view_data
- Signature: `void initialize_view_data(struct view_data *view)`
- Purpose: Set up view structure with default/computed values (screen dimensions, scaling factors, cone angles)
- Inputs: Pointer to uninitialized `view_data`
- Outputs/Return: Populates view structure in-place
- Side effects: Modifies view_data fields
- Calls: (calls not visible; likely uses FOV accessors from ViewControl.h)
- Notes: Per-frame setup; handles standard_screen_width vs. projected width distinction

### render_view
- Signature: `void render_view(struct view_data *view, struct bitmap_definition *destination)`
- Purpose: Main rendering callΓÇötransforms world geometry, performs visibility culling, rasterizes to framebuffer
- Inputs: Configured `view_data`; destination bitmap
- Outputs/Return: Framebuffer written to `destination`
- Side effects: Populates render flags buffer, modifies effect_phase counter
- Calls: (not visible in header)
- Notes: Frame-by-frame entry point

### start_render_effect
- Signature: `void start_render_effect(struct view_data *view, short effect)`
- Purpose: Trigger a transient effect (e.g., fold-in, explosion) and initialize effect_phase
- Inputs: effect enum (`_render_effect_fold_in`, etc.)
- Outputs/Return: Sets viewΓåÆeffect and viewΓåÆeffect_phase
- Side effects: Modifies view_data effect fields
- Calls: (not visible)
- Notes: Effects are animate via effect_phase counter during render_view calls

### instantiate_rectangle_transfer_mode / instantiate_polygon_transfer_mode
- Signature: Both take view, shape definition, transfer_mode, and phase/horizontal flag
- Purpose: Apply texture/color transfer modes (tinting, static, landscape, etc.) to sprites/polygons
- Inputs: view_data, geometry definition, transfer mode enum, mode-specific parameter
- Outputs/Return: None (modify definition in-place or apply to view state)
- Side effects: Updates shading/color tables in geometry
- Calls: (not visible)
- Notes: Support modes defined in `scottish_textures.h`

## Control Flow Notes

- **Init phase**: `allocate_render_memory()` once, then `initialize_view_data()` per view setup
- **Frame loop**: `render_view()` each frame transforms worldΓåÆscreen, performs frustum culling (sets render flags), and rasterizes
- **Effects**: `start_render_effect()` initiates (e.g., teleport fold); effect_phase counter auto-increments during render calls to animate
- **Conditional UI**: `render_overhead_map()` and `render_computer_interface()` called separately based on `view_dataΓåÆoverhead_map_active` and `terminal_mode_active` flags
- **Transfer modes**: Applied on-demand via `instantiate_*_transfer_mode()` for dynamic shading/texture effects

## External Dependencies

- **world.h**: world coordinates, angles, 3D vectors/points, trig tables
- **textures.h**: `bitmap_definition` structure (frame buffer output format)
- **ViewControl.h**: FOV accessors (`View_FOV_Normal()`, `View_FOV_TunnelVision()`, etc.) and landscape options
- **scottish_textures.h**: `rectangle_definition`, `polygon_definition`, transfer mode enums, tint tables

**Symbols used but defined elsewhere**:
- `_fixed` (fixed-point type from underlying system)
- `angle` (from world.h)
- `world_point3d`, `world_vector2d` (from world.h)
- `bitmap_definition` (from textures.h)
- `uint16`, `uint32`, `short` (platform types)
