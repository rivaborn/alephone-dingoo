# Source_Files/RenderOther/OverheadMap_OGL.cpp

## File Purpose
OpenGL renderer implementation for Marathon's overhead map display. Optimizes map drawing via vertex caching and batch rendering to improve performance at high resolutions, replacing CPU-intensive software rendering.

## Core Responsibilities
- Manage OpenGL state setup/teardown for overhead map rendering
- Cache and batch-render polygons, lines, and paths to reduce draw calls
- Render map primitives: polygons, lines, paths, player symbols, and text
- Handle color and pen-width state changes to minimize OpenGL state switches
- Transform and draw game objects (players, monsters, items) on the map overlay

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `OverheadMap_OGL_Class` | class | Concrete OpenGL implementation inheriting from `OverheadMapClass`; manages all overhead map rendering |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `RenderContext` | `AGLContext` | extern | Mac platform OpenGL context for font rendering; defined in OGL_Render.c |
| `ViewWidth`, `ViewHeight` | `short` | extern | Viewport dimensions for clearing; defined in OGL_Render.cpp |

Members (instance state, not global):
- `PolygonCache`, `LineCache` ΓÇô `vector<unsigned short>` ΓÇô batched vertex indices awaiting render
- `SavedColor`, `SavedPenSize` ΓÇô cached OpenGL state to detect redundant state changes

## Key Functions / Methods

### begin_overall
- **Purpose:** Initialize OpenGL state for overhead map rendering frame.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Clears screen with black polygon; disables depth test, alpha test, blending, textures, fog, culling; sets line width to 1.
- **Notes:** Called once at frame start. Blanking is done via polygon fill rather than glClear (see commented code).

### end_overall
- **Purpose:** Restore OpenGL client state after overhead map rendering.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Re-enables texture coordinate arrays; resets line width to 1.

### begin_polygons / end_polygons
- **Purpose:** Frame batch polygon rendering; reset color cache and clear polygon queue.
- **Calls (begin_polygons):** `glVertexPointer`, `GetVertexStride`, `GetFirstVertex`.
- **Notes:** Vertex arrays enabled conditionally (`USE_VERTEX_ARRAYS`). Triangle-fan tessellation converts polygons to triangles for compatibility.

### draw_polygon
- **Purpose:** Add a polygon to the cache or render immediately if color changes.
- **Inputs:** `vertex_count` (short), `vertices` (short array), `color` (rgb_color&).
- **Outputs/Return:** None.
- **Side effects:** Flushes cached polygons if color differs; updates SavedColor; adds triangle indices to PolygonCache.
- **Notes:** Converts polygons to triangle fans (k=2 to vertex_count). Falls back to immediate mode if vertex arrays disabled.

### DrawCachedPolygons / DrawCachedLines
- **Purpose:** Flush batched geometry to GPU.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Calls `glDrawElements` with cached indices; clears cache.
- **Notes:** Uses `GL_TRIANGLES` for polygons, `GL_LINES` for line segments.

### draw_line
- **Purpose:** Add a line segment to cache or render immediately if color/width changes.
- **Inputs:** `vertices` (short array, 2 indices), `color` (rgb_color&), `pen_size` (short).
- **Outputs/Return:** None.
- **Side effects:** Flushes if color or pen width differ; caches or draws immediately.

### draw_thing
- **Purpose:** Render a simple map symbol (rectangle or octagon circle).
- **Inputs:** `center` (world_point2d&), `color` (rgb_color&), `shape` (short enum), `radius` (short).
- **Outputs/Return:** None.
- **Side effects:** Transforms (translate, scale) via matrix stack; calls `glDrawArrays`.
- **Notes:** Shapes are hardcoded vertex arrays. Circle is approximated as octagon.

### draw_player
- **Purpose:** Render player triangle with rotation to show facing direction and perimeter highlight.
- **Inputs:** `center`, `facing` (angle), `color`, `shrink` (scale), `front`, `rear`, `rear_theta`.
- **Outputs/Return:** None.
- **Side effects:** Builds triangle vertex array; applies rotation and scale; draws filled polygon and line perimeter.
- **Notes:** Triangle computed from front/rear distances and rear angle. Perimeter (line loop) improves visibility at small scale.

### draw_text
- **Purpose:** Render overhead map text annotation at a location.
- **Inputs:** `location` (world_point2d&), `color` (rgb_color&), `text` (char*), `FontData` (FontSpecifier&), `justify` (short: 0=left, 1=center).
- **Outputs/Return:** None.
- **Side effects:** Adjusts location based on text width and justification; calls `FontData.OGL_Render`.
- **Notes:** Uses FontSpecifier (platform-specific font handler). Pushes/pops matrix for position.

### set_path_drawing / draw_path / finish_path
- **Purpose:** Batch accumulate and render a continuous path (e.g., monster patrol route).
- **Inputs (draw_path):** `step` (int, 0 = reset), `location` (world_point2d&).
- **Outputs/Return:** None.
- **Side effects:** Clears PathPoints on step Γëñ 0; appends location; finish_path calls `glDrawArrays(GL_LINE_STRIP)`.
- **Notes:** Efficient line-strip batching. FontSpecifier::TextWidth used for text layout.

## Control Flow Notes
Frame-based pipeline: 
1. `begin_overall()` ΓåÆ setup state
2. Sequence of `begin_*()`, `draw_*()`, `end_*()` triplets (polygons, lines, paths, objects, text)
3. `end_overall()` ΓåÆ restore state

Caching flushes on state change (color, pen width) or at phase boundary to reduce redundant OpenGL calls. Vertex arrays (`USE_VERTEX_ARRAYS`) enabled by default for batch efficiency; fallback to immediate mode available.

## External Dependencies
- **OpenGL:** `glColor3f`, `glBegin`/`glEnd`, `glVertex*`, `glClear`, `glLineWidth`, `glMatrixMode`, `glPushMatrix`/`glPopMatrix`, `glTranslatef`, `glRotatef`, `glScalef`, `glVertexPointer`, `glDrawArrays`, `glDrawElements`, `glEnable`/`glDisable`, `glEnableClientState`/`glDisableClientState`
- **Game types/functions (defined elsewhere):** `world_point2d`, `world_point3d`, `rgb_color`, `angle`, `GetVertex`, `GetVertexStride`, `GetFirstVertex`, `FontSpecifier::TextWidth`, `FontSpecifier::OGL_Render`, `FULL_CIRCLE`
- **Utilities:** `SglColor3usv` (color wrapper from OGL_Setup.h); `SetColor`, `ColorsEqual` (inline helpers)
- **Platform:** `RenderContext` (Mac AGL context), `ViewWidth`/`ViewHeight` globals
