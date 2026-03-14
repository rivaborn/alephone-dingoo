# Source_Files/Misc/Console.cpp

## File Purpose
Implements an interactive console system for Aleph One with command parsing, macro expansion, and networked kill message reporting. Provides both in-game user-facing console input and internal command routing infrastructure, along with XML configuration support for macros and carnage messages.

## Core Responsibilities
- Command registration and dispatch via CommandParser (supports hierarchical subcommands)
- Interactive text input handling (buffering, display, keyboard events)
- Macro expansion and substitution before command execution
- Kill message reporting in networked multiplayer games with template customization
- Level saving command implementation with filename persistence
- XML parsing and configuration for console settings, macros, and kill messages
- Singleton lifecycle management for the Console instance

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `CommandParser` | class | Base class for command registration/execution; maps command strings to handlers |
| `Console` | class | Singleton console manager extending CommandParser; handles input/output |
| `XML_CarnageMessageParser` | class | Parses XML `<carnage_message>` elements for kill message templates |
| `XML_MacroParser` | class | Parses XML `<macro>` elements defining text substitution aliases |
| `XML_ConsoleParser` | class | Parses XML `<console>` element for Lua console setting |
| `save_level` | struct (functor) | Implements level export command with filename/history logic |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `Console::m_instance` | `Console*` | static | Singleton instance pointer |
| `CarnageMessageParser` | `XML_CarnageMessageParser` | static | XML parser for kill messages; reused across parses |
| `MacroParser` | `XML_MacroParser` | static | XML parser for macros; reused across parses |
| `ConsoleParser` | `XML_ConsoleParser` | static | XML parser for console settings; reused across parses |
| `last_level` | `std::string` | static | Persistent cache of last saved level filename (cleared on level reset) |
| `game_is_networked` | `bool` | global (external) | Indicates if game is running in networked mode |

## Key Functions / Methods

### Console::instance()
- **Signature:** `static Console* instance()`
- **Purpose:** Lazy-initialized singleton accessor
- **Inputs:** None
- **Outputs/Return:** Pointer to the sole Console instance
- **Side effects:** Creates instance on first call
- **Calls:** `new Console`
- **Notes:** Not thread-safe; assumes single-threaded initialization

### CommandParser::register_command (overload 1: function handler)
- **Signature:** `void register_command(string command, boost::function<void(const string&)> f)`
- **Purpose:** Registers a command string mapped to a callback function
- **Inputs:** `command` (normalized to lowercase), function object `f` taking remainder string
- **Outputs/Return:** None
- **Side effects:** Inserts into `m_commands` map
- **Calls:** `lowercase()`
- **Notes:** Supports nested CommandParser objects as command handlers via overload 2

### CommandParser::parse_and_execute
- **Signature:** `void parse_and_execute(const std::string& command_string)`
- **Purpose:** Splits input into command and remainder, looks up handler, invokes it
- **Inputs:** Full command string (e.g., "save level.sceA")
- **Outputs/Return:** None
- **Side effects:** Executes matched command handler; no output if command not found
- **Calls:** `split()`, `lowercase()`, `m_commands.find()`
- **Notes:** Silently ignores unknown commands; only first space-delimited token is command

### Console::activate_input
- **Signature:** `void activate_input(boost::function<void(const string&)> callback, const string& prompt)`
- **Purpose:** Enters interactive input mode with provided callback and prompt text
- **Inputs:** `callback` to invoke on Enter; `prompt` to display
- **Outputs/Return:** None
- **Side effects:** Sets `m_active = true`, clears buffers, enables SDL key repeat/Unicode if SDL build
- **Calls:** SDL_EnableKeyRepeat, SDL_EnableUNICODE (if defined)
- **Notes:** Asserts `!m_active` (no nested activation); `callback` must remain valid until `enter()` or `abort()` is called

### Console::enter
- **Signature:** `void enter()`
- **Purpose:** Processes accumulated input buffer; macro expansion ΓåÆ command dispatch ΓåÆ callback
- **Inputs:** None (uses `m_buffer` accumulated via `key()` calls)
- **Outputs/Return:** None
- **Side effects:** Expands macros, executes command (if buffer starts with '.'), or invokes callback; clears buffers, deactivates input, disables SDL repeat/Unicode
- **Calls:** `split()`, `lowercase()`, `parse_and_execute()`, `m_callback()`
- **Notes:** Macros checked first (if buffer starts with '.'); command-style input (starting with '.') overrides callback

### Console::report_kill
- **Signature:** `void report_kill(int16 player_index, int16 aggressor_player_index, int16 projectile_index)`
- **Purpose:** Displays kill message in networked games using configured carnage message templates
- **Inputs:** Victim player index, aggressor player index, projectile that caused kill
- **Outputs/Return:** None
- **Side effects:** Calls `screen_printf()` to display message on HUD
- **Calls:** `get_projectile_data()`, `get_player_data()`, `replace_first()`, `NetAllowCarnageMessages()`, `screen_printf()`
- **Notes:** Only displays if `game_is_networked && NetAllowCarnageMessages() && m_carnage_messages_exist`; supports `%player%` and `%aggressor%` placeholders in templates; order of placeholder replacement matters (checked via position comparison)

### XML_CarnageMessageParser::AttributesDone
- **Signature:** `bool AttributesDone()`
- **Purpose:** Finalizes parsed carnage message and registers it with Console singleton
- **Inputs:** None (uses member vars `projectile_type`, `on_kill`, `on_suicide` set by `HandleAttribute()`)
- **Outputs/Return:** `true` if validation passed, `false` if `projectile_type == NONE`
- **Side effects:** Calls `Console::instance()->set_carnage_message()`
- **Calls:** `Console::instance()`, `set_carnage_message()`
- **Notes:** Validates projectile_type is defined; empty strings for messages are allowed

## Control Flow Notes
**Initialization**: XML configuration (`Console_GetParser()`) is parsed at startup, populating macros and carnage messages.

**Interactive Input**: User activates console via `activate_input()` ΓåÆ game loop calls `key()/backspace()` ΓåÆ accumulates in `m_buffer` ΓåÆ user presses Enter ΓåÆ `enter()` processes ΓåÆ clears and deactivates.

**Macro Expansion**: On `enter()`, if buffer starts with '.', the next token is looked up in `m_macros` and substituted.

**Command Dispatch**: If buffer starts with '.', `parse_and_execute()` is called; otherwise callback is invoked (no command routing).

**Carnage Reporting**: During gameplay, `report_kill()` is called when a kill occurs in a networked game; it formats and displays the kill message using template substitution, gated by networking and permission checks.

**Level Saving**: Hierarchical command structure allows `.save level [filename]` routing through SaveCommandParser.

## External Dependencies
- **Notable includes:**
  - `cseries.h`: Core engine utilities (string, macros)
  - `Console.h`: Console interface definition
  - `Logging.h`: `logAnomaly()` for error reporting
  - `network.h`: `game_is_networked`, `NetAllowCarnageMessages()`, `NetAllowSavingLevel()`
  - `player.h`: `get_player_data()` for player names
  - `projectiles.h`: `get_projectile_data()` for projectile types
  - `shell.h`: `screen_printf()` for HUD output
  - `FileHandler.h`: `FileSpecifier` for level export
  - `game_wad.h`: `export_level()`, `static_world`, `mac_roman_to_utf8()`
  - Boost: `bind`, `function` (for command handler callbacks)
  - STL: `string`, `map`, `vector`, `pair`, `algorithm`
  
- **Functions/symbols defined elsewhere:**
  - `export_level()`, `static_world`, `mac_roman_to_utf8()`, `utf8_to_mac_roman()` (game_wad)
  - `get_player_data()`, `get_projectile_data()` (player/projectile subsystems)
  - `screen_printf()` (shell/display)
  - `NetAllowCarnageMessages()`, `NetAllowSavingLevel()`, `game_is_networked` (network)
  - XML parser base classes and utilities (XML_ElementParser from framework)
