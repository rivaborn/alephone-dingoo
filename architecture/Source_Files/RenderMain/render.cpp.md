# Source_Files/RenderMain/render.cpp

## File Purpose
Central coordinator of the Marathon engine's 3D rendering pipeline. Orchestrates visibility determination, polygon depth-sorting, object placement, and rasterization. Manages view camera configuration (FOV, position, orientation) and applies visual effects (explosions, teleport folds, transfer modes).

## Core Responsibilities
- Coordinate visibility-tree construction, polygon sorting, object placement, and rasterization via composition of specialized classes
- Initialize and update view data (camera position, FOV, clipping planes, pitch/yaw/roll)
- Apply render effects (fold in/out, explosions) by modifying view parameters
- Instantiate transfer modes (invisibility, fading, static, pulsating, wobbling) for sprites and surfaces
- Render the foreground weapon/HUD layer
- Allocate and initialize render subsystem memory structures
- Manage render flags for vertices, lines, and polygons (visibility, transformation, clipping)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `view_data` | struct | Camera/viewport state: position, orientation, FOV, clipping planes, effects phase |
| `rectangle_definition` | struct (external) | Texture-mapped sprite rectangle with clipping, transfer mode, lighting |
| `polygon_definition` | struct (external) | Surface geometry with texture, transfer mode, origin/vector for animation |
| `weapon_display_information` | struct | HUD weapon sprite metadata: position, frame, collection, transfer mode |
| `RenderVisTreeClass` | class | Builds visibility tree (which polygons visible from which) |
| `RenderSortPolyClass` | class | Sorts polygons into depth order and accumulates clipping |
| `RenderPlaceObjsClass` | class | Places objects into sorted render order |
| `RenderRasterizerClass` | class | Clips and rasterizes surfaces |
| `Rasterizer_SW_Class` | class | Software rasterizer backend |
| `Rasterizer_OGL_Class` | class | OpenGL rasterizer backend |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `RenderVisTree` | RenderVisTreeClass | static | Visibility tree builder instance |
| `RenderSortPoly` | RenderSortPolyClass | static | Polygon sorter instance |
| `RenderPlaceObjs` | RenderPlaceObjsClass | static | Object placement instance |
| `RenderRasterize` | RenderRasterizerClass | static | Rasterizer controller instance |
| `Rasterizer_SW` | Rasterizer_SW_Class | static | Software rasterizer instance |
| `Rasterizer_OGL` | Rasterizer_OGL_Class | static | OpenGL rasterizer instance |
| `RenderFlagList` | vector\<uint16> | file-global | Per-entity visibility/clipping flags |

## Key Functions / Methods

### allocate_render_memory
- Signature: `void allocate_render_memory(void)`
- Purpose: Initialize all render subsystem memory and interconnect render classes
- Inputs: None
- Outputs/Return: None
- Side effects: Resizes RenderFlagList, calls Resize() on render classes, wires up internal pointers (e.g., RenderSortPoly.RVPtr = &RenderVisTree)
- Calls: RenderVisTree.Resize(), RenderSortPoly.Resize()
- Notes: Called once at startup; asserts that pointer size equals POINTER_DATA size

### initialize_view_data
- Signature: `void initialize_view_data(struct view_data *view)`
- Purpose: Calculate view matrices (screen-to-world mapping, clipping planes) from FOV and screen dimensions
- Inputs: view with field_of_view, screen_width, screen_height, standard_screen_width, vertical/horizontal_scale set
- Outputs/Return: Modifies view with computed world_to_screen_x/y, half_cone, clipping edge vectors
- Side effects: Calculates landscape_yaw for background rendering
- Calls: View_FOV_FixHorizontalNotVertical(), atan(), sin(), asin()
- Notes: Handles non-square aspect ratios; clears active render effects

### render_view
- Signature: `void render_view(struct view_data *view, struct bitmap_definition *destination)`
- Purpose: Main per-frame rendering pipelineΓÇöupdates view, builds visibility tree, sorts, places objects, rasterizes
- Inputs: view (camera state), destination (target framebuffer)
- Outputs/Return: Rendered image written to destination
- Side effects: Clears render flags, updates automap, modifies view internal state
- Calls: update_view_data(), RenderVisTree.build_render_tree(), RenderSortPoly.sort_render_tree(), RenderPlaceObjs.build_render_object_list(), RenderRasterize.render_tree(), render_viewer_sprite_layer(), ResetOverheadMap()
- Notes: Branches on terminal_mode_active, overhead_map_active; selects software or OpenGL rasterizer based on OGL_IsActive()

### update_view_data
- Signature: `static void update_view_data(struct view_data *view)`
- Purpose: Update clipping planes and camera vectors based on current position, orientation, and effects
- Inputs: view with yaw, pitch, origin, media state
- Outputs/Return: Modifies view left/right/top/bottom edges, dtanpitch, landscape_yaw
- Side effects: May inset view->origin by 1 unit if on a vertex (prevents ray-tree degeneracy); updates under_media_boundary
- Calls: View_AdjustFOV(), update_render_effect(), NORMALIZE_ANGLE(), sine_table[], cosine_table[]
- Notes: Calls inset logic to push camera away from polygon vertices to maintain tree consistency

### update_render_effect
- Signature: `static void update_render_effect(struct view_data *view)`
- Purpose: Animate active render effects (explosions, teleport folds) by modifying view FOV and position
- Inputs: view with effect, effect_phase, ticks_elapsed
- Outputs/Return: Modifies view effect, effect_phase, world_to_screen_x/y
- Side effects: None (pure state update)
- Calls: shake_view_origin()
- Notes: Expires effects when phase > period; fold effects scale world_to_screen to create shrinking/expanding

### start_render_effect
- Signature: `void start_render_effect(struct view_data *view, short effect)`
- Purpose: Begin a render effect (used for explosions, teleports)
- Inputs: view, effect (one of _render_effect_*)
- Outputs/Return: Modifies view->effect and effect_phase
- Side effects: None
- Calls: None

### instantiate_rectangle_transfer_mode
- Signature: `void instantiate_rectangle_transfer_mode(view_data *view, rectangle_definition *rectangle, short transfer_mode, _fixed transfer_phase)`
- Purpose: Apply visual transfer mode (invisibility, static, fading, folding) to a sprite rectangle
- Inputs: view (for shading mode), rectangle (sprite to modify), transfer_mode, transfer_phase (0..FIXED_ONE)
- Outputs/Return: Modifies rectangle (transfer_mode, transfer_data, x0/x1 for folds)
- Side effects: None
- Calls: get_global_shading_table()
- Notes: Handles ~13 modes; fold modes scale sprite toward center (xc); infravision treats invisibility as normal

### instantiate_polygon_transfer_mode
- Signature: `void instantiate_polygon_transfer_mode(struct view_data *view, struct polygon_definition *polygon, short transfer_mode, bool horizontal)`
- Purpose: Apply transfer mode effects (slide, wander, pulsate, wobble) to surface geometry by animating origin/vector
- Inputs: view (tick_count for timing), polygon, transfer_mode, horizontal (floor/ceiling vs. wall)
- Outputs/Return: Modifies polygon origin, vector, transfer_mode
- Side effects: None
- Calls: sine_table[], cosine_table[], isqrt()
- Notes: Slide/wander use sinusoidal animation; pulsate/wobble modify height or perpendicular offset

### render_viewer_sprite_layer
- Signature: `static void render_viewer_sprite_layer(view_data *view, RasterizerClass *RasPtr)`
- Purpose: Render the foreground HUD weapons layer
- Inputs: view (for screen dimensions, shading mode, lighting), RasPtr (rasterizer backend)
- Outputs/Return: Rendered weapon sprites written via RasPtr->texture_rectangle()
- Side effects: Sets foreground rendering state on rasterizer
- Calls: get_weapon_display_information(), extended_get_shape_information(), extended_get_shape_bitmap_and_shading_table(), position_sprite_axis(), instantiate_rectangle_transfer_mode()
- Notes: Skips if view->show_weapons_in_hand is false; supports 3D weapon models via OGL_GetModelData(); applies shape mirroring flags

### position_sprite_axis
- Signature: `static void position_sprite_axis(short *x0, short *x1, short scale_width, short screen_width, short positioning_mode, _fixed position, bool flip, world_distance world_left, world_distance world_right)`
- Purpose: Calculate screen coordinates for a sprite axis given world dimensions and positioning mode
- Inputs: scale_width (view height), screen_width, positioning_mode (_position_low/center/high), position (fixed-point parameter), flip, world dimensions
- Outputs/Return: Modifies *x0, *x1 with screen coordinates
- Side effects: None
- Calls: None
- Notes: Three modes for weapon positioning; handles horizontal flipping by swapping/negating world coordinates

### shake_view_origin
- Signature: `static void shake_view_origin(struct view_data *view, world_distance delta)`
- Purpose: Apply explosion effect by adding sinusoidal jitter to camera position
- Inputs: view, delta (amplitude of shake)
- Outputs/Return: Modifies view->origin if new position stays in same polygon
- Side effects: Reads view->tick_count for phase; calls find_line_crossed_leaving_polygon()
- Calls: sine_table[], find_line_crossed_leaving_polygon()
- Notes: Uses different frequencies for x/y/z to create natural turbulence; only applies shake if camera doesn't cross polygon boundary

## Control Flow Notes
**Initialization:** `allocate_render_memory()` wires together render subsystems and allocates buffers.

**Per-frame rendering** (`render_view()`):
1. Update view camera vectors and clipping planes (`update_view_data()`)
2. Clear render flags
3. Branch on mode:
   - **Terminal/interface mode:** render computer UI
   - **Overhead map mode:** render minimap
   - **Normal 3D mode:**
     - Build visibility tree (which polygons visible)
     - Sort polygons by depth
     - Place visible objects in depth order
     - Select software or OpenGL rasterizer
     - Clip and rasterize each surface
     - Render HUD weapons layer

**Effects:** Active effects (explosions, teleport folds) are updated per-frame in `update_render_effect()`, which modifies FOV or camera shake until expiration.

## External Dependencies
- **map.h:** Polygon, line, endpoint, object, lightsource data structures
- **render.h:** view_data, render flags
- **lightsource.h:** Light intensity queries
- **media.h:** Liquid/media boundary checks
- **weapons.h:** Weapon display info, animation frames
- **interface.h:** Shape information, bitmap/shading table access
- **ViewControl.h:** FOV adjustment accessors
- **AnimatedTextures.h:** Animated texture support
- **OGL_Render.h:** OpenGL rasterizer interface
- **RenderVisTree.h, RenderSortPoly.h, RenderPlaceObjs.h, RenderRasterize.h:** Decomposed render subsystems
- **Rasterizer_SW.h, Rasterizer_OGL.h:** Rasterizer implementations
- **cmath, cstring, cstdlib:** Standard library (math, strings, memory)
