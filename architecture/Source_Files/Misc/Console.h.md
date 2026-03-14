# Source_Files/Misc/Console.h

## File Purpose
Defines a singleton console system for Aleph One that manages command parsing, user input handling, and carnage reporting. Provides command registration, macro expansion, and optional Lua console integration for in-game scripting and debugging.

## Core Responsibilities
- Register and execute string-based console commands via `CommandParser` base class
- Manage active/inactive console input state with user-provided callbacks
- Handle keyboard input (key presses, backspace, enter, abort, clear)
- Store and expand user-defined macros
- Track and report player kills with customizable kill/suicide messages
- Integrate with Lua console mode for scripting
- Persist and manage save game commands

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `CommandParser` | class | Base class for hierarchical command parsing and registration |
| `Console` | class | Singleton console instance extending `CommandParser` |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `Console::m_instance` | `Console*` | static | Singleton instance pointer |

## Key Functions / Methods

### CommandParser::register_command
- **Signature:** `void register_command(std::string command, boost::function<void(const std::string&)> f)` and overload taking `const CommandParser&`
- **Purpose:** Register a named command with a callback function or delegate to another `CommandParser`
- **Inputs:** Command name string, callback function or another `CommandParser`
- **Outputs/Return:** None
- **Side effects:** Modifies `m_commands` map
- **Calls:** None visible
- **Notes:** Supports command delegation to nested parsers

### CommandParser::unregister_command
- **Signature:** `void unregister_command(std::string command)`
- **Purpose:** Remove a registered command
- **Inputs:** Command name
- **Outputs/Return:** None
- **Side effects:** Removes entry from `m_commands` map
- **Calls:** None visible

### CommandParser::parse_and_execute
- **Signature:** `virtual void parse_and_execute(const std::string& command_string)`
- **Purpose:** Parse and execute a command string
- **Inputs:** Command string
- **Outputs/Return:** None
- **Side effects:** Invokes registered callback or nested parser
- **Calls:** Registered callbacks or nested `CommandParser::parse_and_execute`
- **Notes:** Virtual; may be overridden by subclasses

### Console::instance
- **Signature:** `static Console* instance()`
- **Purpose:** Return singleton instance
- **Inputs:** None
- **Outputs/Return:** Pointer to singleton `Console`
- **Side effects:** Lazy-initializes singleton if needed
- **Calls:** None visible in header

### Console::activate_input / deactivate_input
- **Signature:** `void activate_input(boost::function<void (const std::string&)> callback, const std::string& prompt)` / `void deactivate_input()`
- **Purpose:** Enable/disable console input mode with a callback for when user presses enter
- **Inputs:** Callback function and prompt string
- **Outputs/Return:** None
- **Side effects:** Sets `m_active`, stores callback and prompt, displays prompt
- **Calls:** None visible
- **Notes:** `deactivate_input()` aborts without invoking callback; `input_active()` returns current state

### Console::enter / abort / backspace / clear / key
- **Signature:** `void enter()`, `void abort()`, `void backspace()`, `void clear()`, `void key(const char)`
- **Purpose:** Handle keyboard input during active console input
- **Inputs:** Keyboard character (for `key()`)
- **Outputs/Return:** None
- **Side effects:** Modify `m_buffer` and `m_displayBuffer`; `enter()` invokes callback with buffer; `abort()` invokes callback with empty string
- **Calls:** None visible
- **Notes:** These are typically driven by input event handlers

### Console::register_macro / unregister_macro
- **Signature:** `void register_macro(std::string macro, std::string replacement)` / `void unregister_macro(std::string macro)`
- **Purpose:** Register/remove text expansion macros
- **Inputs:** Macro name and replacement text
- **Outputs/Return:** None
- **Side effects:** Modifies `m_macros` map
- **Calls:** None visible

### Console::set_carnage_message
- **Signature:** `void set_carnage_message(int16 projectile_type, const std::string& on_kill, const std::string& on_suicide = "")`
- **Purpose:** Register kill/suicide message for a projectile type
- **Inputs:** Projectile type index, kill message, optional suicide message
- **Outputs/Return:** None
- **Side effects:** Appends to `m_carnage_messages` vector; sets `m_carnage_messages_exist` flag
- **Calls:** None visible

### Console::report_kill
- **Signature:** `void report_kill(int16 player_index, int16 aggressor_player_index, int16 projectile_index)`
- **Purpose:** Record a kill and potentially display associated message
- **Inputs:** Victim player index, killer player index, projectile index
- **Outputs/Return:** None
- **Side effects:** Looks up and potentially displays carnage message for projectile; updates game state
- **Calls:** Likely game engine functions (not visible in header)

## Control Flow Notes
- `Console` is a **singleton**; typical usage: `Console::instance()->ΓÇª` 
- Command execution flow: `parse_and_execute()` ΓåÆ lookup in `m_commands` map ΓåÆ invoke callback or delegate to nested parser
- Input flow: Keyboard events ΓåÆ `key()` / `enter()` / etc. ΓåÆ modifies `m_buffer` ΓåÆ callback invoked on `enter()`
- Macro expansion occurs during `parse_and_execute()` (implementation not visible)
- Carnage reporting integrated into game loop (likely called by damage/kill handler)

## External Dependencies
- **Standard Library:** `<string>`, `<map>`, `<vector>`
- **Boost:** `boost::function<>` for type-erased function callbacks
- **Engine headers:** 
  - `XML_ElementParser.h` ΓÇô for XML preferences parsing (`Console_GetParser()`)
  - `preferences.h` ΓÇô accesses `environment_preferences->use_solo_lua`
- **Defined elsewhere:** `Console_GetParser()` function, `environment_preferences` global
