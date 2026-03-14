# Source_Files/Misc/progress.h

## File Purpose
Header file defining the progress dialog interface for the game engine. Provides UI feedback during long-running operations such as network synchronization, map distribution, physics loading, and router port management.

## Core Responsibilities
- Define message ID constants for different progress states (network transfers, loading, router operations)
- Manage progress dialog lifecycle (open/close)
- Update dialog messages and progress bar visualization
- Abstract progress UI implementation details

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| (unnamed enum) | enum | Message IDs and state constants for progress dialog states |

## Global / File-Static State
None.

## Key Functions / Methods

### open_progress_dialog
- Signature: `void open_progress_dialog(size_t message_id, bool show_progress_bar = false)`
- Purpose: Opens and displays the progress dialog with the specified message.
- Inputs: `message_id` (string resource ID from enum), `show_progress_bar` (optional, default false)
- Outputs/Return: None
- Side effects: Creates/shows UI dialog; may allocate display resources
- Notes: Default parameter suggests C++ implementation or wrapper; `show_progress_bar` controls whether progress bar is rendered

### close_progress_dialog
- Signature: `void close_progress_dialog(void)`
- Purpose: Closes and dismisses the progress dialog.
- Inputs: None
- Outputs/Return: None
- Side effects: Destroys/hides UI dialog; releases display resources

### set_progress_dialog_message
- Signature: `void set_progress_dialog_message(size_t message_id)`
- Purpose: Updates the message displayed in the currently-open dialog.
- Inputs: `message_id` (new string resource ID)
- Outputs/Return: None
- Side effects: Updates displayed UI text

### draw_progress_bar
- Signature: `void draw_progress_bar(size_t sent, size_t total)`
- Purpose: Renders or updates the progress bar with sent/total bytes transferred.
- Inputs: `sent` (bytes transferred), `total` (total bytes to transfer)
- Outputs/Return: None
- Side effects: Updates progress bar UI state; typically called per frame during transfer

### reset_progress_bar
- Signature: `void reset_progress_bar(void)`
- Purpose: Clears progress bar state back to initial/empty state.
- Inputs: None
- Outputs/Return: None
- Side effects: Resets progress bar display

## Control Flow Notes
Typical usage pattern: `open_progress_dialog()` ΓåÆ `set_progress_dialog_message()` + `draw_progress_bar()` (loop) ΓåÆ `close_progress_dialog()`. Primarily used during:
- **Network operations**: Map and physics distribution (single/multiple clients), receiving assets
- **Local operations**: File loading
- **Router management**: Opening/closing ports
- **Updates**: Version checking

## External Dependencies
- C standard library: `size_t`, `bool` type
- Resource strings: `strPROGRESS_MESSAGES` (resource ID 143) containing localized dialog messages
- Implementation elsewhere: actual dialog rendering and string loading functions
