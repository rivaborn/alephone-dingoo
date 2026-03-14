# Source_Files/RenderOther/computer_interface.h

## File Purpose
Defines the interface for the in-game computer terminal system, which renders and manages interactive terminal displays for players (briefing text, status messages, menus). Handles player input during terminal interaction, terminal state serialization, and text preprocessing/formatting.

## Core Responsibilities
- Initialize and manage terminal mode per player
- Render terminal display with formatted text, embedded formatting codes, and media references
- Process player input and actions while in terminal mode  
- Serialize/deserialize terminal state for persistence across map/level loads
- Preprocess and encode terminal text resources (at build time)
- Manage terminal data packing (player state vs. map-embedded data)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `static_preprocessed_terminal_data` | struct | Metadata for preprocessed terminal text: grouping count, font changes, text length |
| `view_terminal_data` | struct | Display region and scroll offset for rendering terminal on-screen |

## Global / File-Static State
None.

## Key Functions / Methods

### initialize_terminal_manager
- Signature: `void initialize_terminal_manager(void)`
- Purpose: System-level setup for terminal manager subsystem
- Calls: Not visible in header

### enter_computer_interface
- Signature: `void enter_computer_interface(short player_index, short text_number, short completion_flag)`
- Purpose: Transition player into terminal mode, load and display specified terminal text
- Inputs: player index, terminal text ID, completion flag (likely for tracking viewed terminals)
- Side effects: Switches player state; triggers text load and render

### _render_computer_interface
- Signature: `void _render_computer_interface(struct view_terminal_data *data)`
- Purpose: Render current terminal display to screen
- Inputs: view region and scroll data
- Calls: Rendering backend (not visible)

### update_player_for_terminal_mode
- Signature: `void update_player_for_terminal_mode(short player_index)`
- Purpose: Per-frame state update for player in terminal mode
- Side effects: Updates player position, physics, or animations while terminal is open

### update_player_keys_for_terminal
- Signature: `void update_player_keys_for_terminal(short player_index, uint32 action_flags)`
- Purpose: Process player input (scroll, select, navigate) within terminal UI
- Inputs: player index, action flags from input system

### player_in_terminal_mode
- Signature: `bool player_in_terminal_mode(short player_index)`
- Purpose: Query whether player is currently viewing a terminal
- Return: True if in terminal mode

### pack_player_terminal_data / unpack_player_terminal_data
- Signature: `uint8 *pack_player_terminal_data(uint8 *Stream, size_t Count)` | `uint8 *unpack_player_terminal_data(uint8 *Stream, size_t Count)`
- Purpose: Serialize/deserialize per-player terminal state (e.g., viewed terminals, scroll position)
- Return: Pointer to next byte in stream

**Notes:** Trivial helpers include `dirty_terminal_view()` (mark for redraw), `abort_terminal_mode()` (exit terminal), `calculate_packed_terminal_data_length()` (size query), `build_terminal_action_flags()` (map keymap to action flags).

## Control Flow Notes
Typical frame flow:
1. Player enters terminal via `enter_computer_interface(text_id, ...)`
2. Per-frame loop: `update_player_for_terminal_mode()` ΓåÆ `_render_computer_interface()`
3. Input processed via `update_player_keys_for_terminal(action_flags)` each frame
4. Player exits via `abort_terminal_mode()` 

Terminal state persists via pack/unpack functions (likely called on level load/save or net sync).

## External Dependencies
- Standard fixed-width integer types (`int16`, `uint32`, `uint8`, `bool`, `byte`) ΓÇö defined elsewhere
- `#ifdef PREPROCESSING_CODE` gates build-time text processing functions (used by offline tool)
- GPL license header indicates Aleph One / Marathon engine origin
