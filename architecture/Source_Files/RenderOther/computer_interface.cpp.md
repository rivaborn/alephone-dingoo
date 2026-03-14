# Source_Files/RenderOther/computer_interface.cpp

## File Purpose
Manages in-game computer terminal interfaces, including rendering, input handling, and state tracking. Handles terminal text display with rich formatting (colors, styles), group transitions (logon/information/teleport), and serialization of terminal data loaded from maps.

## Core Responsibilities
- Initialize and maintain per-player terminal state machines
- Load and unpack terminal data from maps (groupings, text, style changes)
- Render terminal screens with borders, text, images (pictures/videos), and checkpoints
- Parse player input (arrow keys, page up/down, space, escape) during terminal mode
- Calculate text layout (line breaks, word wrapping) based on font metrics
- Serialize/deserialize terminal state for save/load operations
- Handle terminal group transitions (logon ΓåÆ information ΓåÆ teleport sequences)
- Support Lua callbacks for terminal entry/exit events
- Manage text styling (bold, italic, underline, color) via embedded markup

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `terminal_text_t` | struct | One terminal object: groupings, font changes, raw text buffer, copy semantics |
| `player_terminal_data` | struct | Per-player terminal state: current group/line, phase, mode flags |
| `terminal_groupings` | struct | A logical group within terminal (logon, info, checkpoint, etc.); type, permutation, text range |
| `text_face_data` | struct | Style change point: index in text, face flags (bold/italic/underline), color index |
| `view_terminal_data` | struct | Rendering bounds (top, left, bottom, right) + vertical scroll offset |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `map_terminal_text` | `vector<terminal_text_t>` | static | All terminals loaded from current map |
| `player_terminals` | `player_terminal_data*` | static | Per-player state array; allocated at init |
| `current_pixel` | `uint32` | static | SDL pixel value for current text color |
| `current_style` | `uint16` | static | SDL font style flags (bold/italic/underline) |
| `terminal_keys[]` | `terminal_key[]` | static | KeyΓåÆaction mappings (SDL keycodes) |

## Key Functions / Methods

### initialize_terminal_manager
- **Signature:** `void initialize_terminal_manager(void)`
- **Purpose:** One-time setup: allocate player terminal state array, clear all entries, optionally build key code lookup tables (Mac-specific).
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Allocates heap memory for `player_terminals` array (size = `MAXIMUM_NUMBER_OF_PLAYERS`), sets all state to inactive
- **Calls:** `objlist_clear()`, optionally builds Mac key masks
- **Notes:** Called once at game startup; does not load map data

### enter_computer_interface
- **Signature:** `void enter_computer_interface(short player_index, short text_number, short completion_flag)`
- **Purpose:** Transition player into terminal mode; load terminal text, initialize state machine, play logon sound, select first group.
- **Inputs:** 
  - `player_index`: which player (0ΓÇô7)
  - `text_number`: terminal ID to load
  - `completion_flag`: saved level completion state (for checkpoints)
- **Outputs/Return:** None
- **Side effects:** Sets `terminalΓåÆstate = _reading_terminal`, resets `current_line`, `current_group` to 0, plays logon sound, marks terminal dirty for re-render
- **Calls:** `get_indexed_terminal_data()`, `calculate_lines_per_page()`, `next_terminal_group()`, `play_object_sound()`
- **Notes:** Gracefully handles missing terminal data (plays sound and exits); recalculates lines-per-page if display mode changed

### _render_computer_interface
- **Signature:** `void _render_computer_interface(struct view_terminal_data *data)`
- **Purpose:** Draw terminal to screen; dispatch on group type (logon/info/checkpoint/picture) and call appropriate rendering functions.
- **Inputs:** `view_terminal_data*` with rendering bounds and vertical offset
- **Outputs/Return:** None (draws directly to SDL surface)
- **Side effects:** Sets clipping rectangle, sets dirty flag to false after rendering, may adjust scroll bounds
- **Calls:** `get_indexed_terminal_data()`, `get_indexed_grouping()`, `draw_terminal_borders()`, `draw_logon_text()`, `draw_computer_text()`, `present_checkpoint_text()`, `display_picture_with_text()`, `fill_terminal_with_static()`
- **Notes:** Checks if terminal is dirty before rendering (optimization); early return if no terminal data or state is inactive; handles checkpoint overflow gracefully

### update_player_for_terminal_mode
- **Signature:** `void update_player_for_terminal_mode(short player_index)`
- **Purpose:** Update terminal state once per game tick; decrement phase timer and auto-advance to next group if timer expires.
- **Inputs:** `player_index`
- **Outputs/Return:** None
- **Side effects:** Decrements `terminalΓåÆphase`, may call `next_terminal_group()` if phase expires
- **Calls:** `get_player_terminal_data()`, `get_indexed_terminal_data()`, `next_terminal_group()`
- **Notes:** Phase timer is used for logon/logoff sequences; non-zero phase blocks user interaction until it reaches 0

### update_player_keys_for_terminal
- **Signature:** `void update_player_keys_for_terminal(short player_index, uint32 action_flags)`
- **Purpose:** Route keyboard input to terminal input handler when player is in terminal mode.
- **Inputs:** `player_index`, `action_flags` (bitfield of pressed keys)
- **Outputs/Return:** None
- **Side effects:** Calls `handle_reading_terminal_keys()` which modifies `current_line`, `current_group`, or exits terminal
- **Calls:** `get_player_terminal_data()`, `handle_reading_terminal_keys()`
- **Notes:** Only processes input in `_reading_terminal` state; otherwise is a no-op

### player_in_terminal_mode
- **Signature:** `bool player_in_terminal_mode(short player_index)`
- **Purpose:** Query whether a player is currently interacting with a terminal.
- **Inputs:** `player_index`
- **Outputs/Return:** `true` if `terminalΓåÆstate != _no_terminal_state`
- **Side effects:** None
- **Calls:** `get_player_terminal_data()`
- **Notes:** Simple state check; used for HUD updates and game logic gating

### calculate_line
- **Signature:** `static bool calculate_line(char *base_text, short width, short start_index, short text_end_index, short *end_index)`
- **Purpose:** Determine where one line of terminal text ends given a pixel width constraint; perform word-wrapping at spaces.
- **Inputs:** 
  - `base_text`: text buffer
  - `width`: available pixel width
  - `start_index`, `text_end_index`: range to measure
- **Outputs/Return:** 
  - `*end_index`: index after last character of this line
  - Returns `true` if end of text reached
- **Side effects:** Reads font metrics from current interface font and style
- **Calls:** `GetInterfaceFont()`, `char_width()`
- **Notes:** Breaks on space or explicit line-end (`MAC_LINE_END` = 13); if no space found, breaks at the width boundary

### unpack_map_terminal_data
- **Signature:** `void unpack_map_terminal_data(uint8 *p, size_t count)`
- **Purpose:** Deserialize terminal data from map file into `map_terminal_text` vector; unpacks headers, groupings, font changes, and text.
- **Inputs:** 
  - `p`: byte stream pointer (binary data from map)
  - `count`: number of bytes available
- **Outputs/Return:** None; populates global `map_terminal_text`
- **Side effects:** Clears existing terminals, allocates new text buffers (heap)
- **Calls:** `StreamToValue()`, `StreamToBytes()` (byte packing macros)
- **Notes:** Asserts on structure size mismatches; text buffer is copied (not reference to stream)

### pack_map_terminal_data
- **Signature:** `void pack_map_terminal_data(uint8 *p, size_t count)`
- **Purpose:** Serialize terminal data from `map_terminal_text` to binary stream for saving.
- **Inputs:** `p` (output buffer), `count` (available space)
- **Outputs/Return:** None; writes to stream
- **Side effects:** None
- **Calls:** `ValueToStream()`, `BytesToStream()` (byte packing macros)
- **Notes:** Mirrors `unpack_map_terminal_data()` structure; assumes caller has pre-allocated buffer

### unpack_player_terminal_data / pack_player_terminal_data
- **Signature:** `uint8 *unpack_player_terminal_data(uint8 *Stream, size_t Count)` / `uint8 *pack_player_terminal_data(uint8 *Stream, size_t Count)`
- **Purpose:** Serialize per-player terminal state (flags, phase, group index, line number, etc.) for save/load.
- **Inputs/Outputs:** Byte stream; returns updated stream pointer
- **Side effects:** Modifies `player_terminals[]` array (unpack) or reads it (pack)
- **Calls:** `StreamToValue()`, `ValueToStream()`
- **Notes:** Processes all `MAXIMUM_NUMBER_OF_PLAYERS` entries sequentially; used during game save/restore

## Control Flow Notes

**Initialization:**
1. `initialize_terminal_manager()` called at game startup
2. `initialize_player_terminal_info(player_index)` called per player at level start

**Per-Tick Update (in game loop):**
1. `update_player_for_terminal_mode(player_index)` ΓÇô decrement timers, auto-transition groups
2. `update_player_keys_for_terminal(player_index, action_flags)` ΓÇô handle input, navigate groups/lines
3. `_render_computer_interface(view_terminal_data*)` ΓÇô render if dirty flag set

**Terminal Entry/Exit:**
- `enter_computer_interface()` ΓÇô triggered by player activating a control panel; initializes state machine
- `abort_terminal_mode()` (defined elsewhere) ΓÇô exit terminal, reset state
- Optional Lua callback `L_Call_Terminal_Exit()` on exit (if `HAVE_LUA`)

**State Machine:**
- States: `_no_terminal_state` (inactive), `_reading_terminal` (active)
- Phase timers used for logon/logoff visual effects; blocks input until phase Γëñ 0
- Current group and line tracked; navigation via next/previous group and scroll keys

**Rendering Pipeline:**
- Clipping rectangle set to terminal window
- Borders drawn first, then content (avoid overdraw bugs)
- Group type determines rendering path: logon ΓåÆ shape, info ΓåÆ text, checkpoint ΓåÆ goal overlay, picture ΓåÆ image + text, etc.

## External Dependencies
- **Notable includes:** `world.h`, `map.h`, `player.h` (game state), `screen_drawing.h` (rendering), `SoundManager.h`, `lua_script.h` (Lua integration), `sdl_fonts.h` (font metrics), `Packing.h` (byte stream utilities), `interface.h` (error strings)
- **External symbols used:** 
  - `dynamic_world`, `static_world` (game state)
  - `get_player_data()`, `get_indexed_terminal_data()` (defined elsewhere)
  - `play_object_sound()`, `Sound_TerminalLogon()` (audio)
  - `calculate_destination_frame()`, `_get_font_line_height()`, `_get_interface_color()` (rendering)
  - `L_Call_Terminal_Exit()` (Lua)
  - `world_pixels`, `draw_surface` (SDL surfaces)
