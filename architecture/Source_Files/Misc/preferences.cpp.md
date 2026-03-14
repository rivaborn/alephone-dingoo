# Source_Files/Misc/preferences.cpp

## File Purpose
Manages user preferences (settings) persistence and UI for the Aleph One game engine. Handles XML-based loading/saving of graphics, input, sound, network, player, and environment preferences, plus provides preference configuration dialogs for each category.

## Core Responsibilities
- **Preference Persistence**: Loads/saves user settings to XML format using a hierarchical parser tree
- **Preference Dialogs**: Creates and manages UI dialogs for player, graphics, sound, input, environment, and crosshair configuration
- **Data Binding**: Bridges UI widgets with preference structures via `Bindable<T>` adapter pattern
- **Validation**: Validates preference values within acceptable ranges
- **System Integration**: Retrieves system username and network availability for defaults
- **Crosshair Customization**: Provides live preview and configuration of crosshair appearance (shape, color, opacity, dimensions)
- **Network Settings**: Manages metaserver login, chat colors, game protocol selection, and port configuration

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `CrosshairPref` | class | Adapter binding crosshair thickness to UI slider (converts 1-based to 0-based index) |
| `ColorComponentPref` | class | Adapter binding color components (16-bit to/from UI range) |
| `OpacityPref` | class | Adapter binding float opacity to 16-level integer slider |
| `XML_CrosshairsPrefsParser` | class | Parses crosshair XML attributes (thickness, shape, color, opacity, etc.) |
| `XML_PlayerPrefsParser` | class | Parses player name, color, team, difficulty, background music settings |
| `XML_MouseButtonPrefsParser` | class | Parses mouse button action bindings |
| `XML_AxisMappingPrefsParser` | class | Parses joystick axis mappings with sensitivity and bounds |
| `XML_KeyPrefsParser` | class | Parses keyboard and SDL key bindings |
| `XML_InputPrefsParser` | class | Parses input device selection, mouse acceleration, joystick settings |
| `XML_SoundPrefsParser` | class | Parses audio channels, volume, music, sample rate, and netmic settings |
| `XML_NetworkPrefsParser` | class | Parses network game type, difficulty, port, protocol, metaserver login, and UPnP settings |
| `XML_EnvironmentPrefsParser` | class | Parses map/physics/shapes file paths, Lua script paths, and display options |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `graphics_preferences` | `graphics_preferences_data*` | global | Current graphics settings |
| `network_preferences` | `network_preferences_data*` | global | Current network settings |
| `player_preferences` | `player_preferences_data*` | global | Current player settings (name, color, crosshair, difficulty) |
| `input_preferences` | `input_preferences_data*` | global | Current input device and key bindings |
| `sound_preferences` | `SoundManager::Parameters*` | global | Current audio settings |
| `environment_preferences` | `environment_preferences_data*` | global | Current map/physics file paths and Lua settings |
| `PrefsInited` | bool | static | Tracks whether preferences system initialized |
| `PrefsRootParser` | `XML_ElementParser` | static | Root of XML parser tree |
| `MarathonPrefsParser` | `XML_ElementParser` | static | Top-level `<mara_prefs>` element parser |
| `sNetworkGameProtocolNames` | `const char*[]` | static | Maps protocol indices to names ("ring", "star") |
| `sPasswordMask` | `const char[]` | static | XOR mask for obfuscating metaserver password in prefs file |

## Key Functions / Methods

### handle_preferences
- **Signature**: `void handle_preferences(void)`
- **Purpose**: Main entry point; displays top-level preferences dialog with buttons for each category (PLAYER, GRAPHICS, SOUND, CONTROLS, ENVIRONMENT)
- **Inputs**: None
- **Outputs/Return**: None
- **Side effects**: Saves current preferences before opening dialog; clears screen; runs modal dialog; redraws main menu afterward
- **Calls**: `write_preferences()`, `clear_screen()`, `display_main_menu()`, `build_8bit_system_color_table()` (if 8-bit mode)
- **Notes**: Saves state before opening so user can cancel and revert

### player_dialog
- **Signature**: `static void player_dialog(void *arg)`
- **Purpose**: Player settings dialog (difficulty, name, color, team, metaserver login/password, custom chat colors, crosshair button)
- **Inputs**: `arg` (parent dialog pointer)
- **Outputs/Return**: None
- **Side effects**: If accepted, updates `player_preferences` and `network_preferences`, calls `write_preferences()`; if cancelled, no changes
- **Calls**: `copy_pstring_to_text_field()`, `copy_pstring_from_text_field()`, `crosshair_dialog()`, various widget accessors
- **Notes**: Detects changes by comparing old/new values; password/login toggles via guest checkbox; color picker widgets manage dependent enable/disable

### crosshair_dialog
- **Signature**: `static void crosshair_dialog(void *arg)`
- **Purpose**: Crosshair customization with live preview; allows adjusting shape, color (RGB), width, gap, size, and opacity
- **Inputs**: `arg` (parent dialog pointer)
- **Outputs/Return**: None
- **Side effects**: On accept, marks crosshair as needing recalculation and saves prefs; on cancel, reverts to original crosshair data
- **Calls**: `crosshair_binders->migrate_all_first_to_second()` (preview update), various slider/widget accessors
- **Notes**: Uses `Bindable<int>` adapters to marshal between UI sliders and preference data; `PreCalced` flag resets to force re-render

### software_rendering_options_dialog
- **Signature**: `static void software_rendering_options_dialog(void* arg)`
- **Purpose**: Graphics options for software rendering (color depth, resolution, alpha blending)
- **Inputs**: `arg` (parent dialog pointer)
- **Outputs/Return**: None
- **Side effects**: Updates `graphics_preferences` if accepted
- **Calls**: Widget constructors and accessors
- **Notes**: Maps depth selections to enum values; only shown when software rendering is active

### XML_CrosshairsPrefsParser::Start / HandleAttribute / End
- **Purpose**: Parses `<crosshairs>` XML element; `Start()` saves current color, `HandleAttribute()` reads attributes, `End()` applies color changes
- **Inputs** (HandleAttribute): `Tag` (attribute name), `Value` (attribute value string)
- **Outputs/Return**: bool (success/failure)
- **Calls**: `ReadInt16Value()`, `ReadFloatValue()`, color utility functions
- **Notes**: Handles color child element via `Color_GetParser()`

### XML_NetworkPrefsParser::HandleAttribute
- **Signature**: `bool XML_NetworkPrefsParser::HandleAttribute(const char *Tag, const char *Value)`
- **Purpose**: Parses network preferences from XML attributes (game type, difficulty, ports, protocols, metaserver credentials, UPnP, update checks)
- **Inputs**: `Tag` (attribute name), `Value` (string value)
- **Outputs/Return**: bool (true if parsed successfully)
- **Side effects**: Updates fields in `network_preferences`; decodes obfuscated password via XOR with `sPasswordMask`
- **Calls**: `ReadBooleanValue()`, `ReadInt16Value()`, `ReadInt32Value()`, `ReadUInt16Value()`, `StringsEqual()`, `DeUTF8_C()`
- **Notes**: Password is XOR-encoded in file for obfuscation (not encryption); handles legacy `metaserver_clear_password` for migration

### SetupPrefsParseTree
- **Signature**: `static void SetupPrefsParseTree()`
- **Purpose**: Builds XML parser hierarchy for all preference elements
- **Inputs**: None
- **Outputs/Return**: None
- **Side effects**: Populates global parser tree with child relationships
- **Calls**: Multiple `AddChild()` on parser instances
- **Notes**: Called once at init; parsers are global singletons

### get_name_from_system
- **Signature**: `static void get_name_from_system(unsigned char *outName)`
- **Purpose**: Retrieves system username for default player name
- **Inputs**: `outName` (buffer for result; assumed large enough)
- **Outputs/Return**: None (result in `outName`)
- **Side effects**: Calls platform-specific APIs (`getlogin()` on Unix, `GetUserName()` on Windows)
- **Calls**: `a1_c2pstr()` (converts C string to Pascal string in-place)
- **Notes**: Falls back to "Bob User" if retrieval fails; on Windows, rejects names with invalid characters

### ethernet_active
- **Signature**: `static bool ethernet_active(void)`
- **Purpose**: Checks if networking is available
- **Inputs**: None
- **Outputs/Return**: bool (always true in this implementation)
- **Side effects**: None
- **Notes**: Stub function; networking availability determined elsewhere

## Control Flow Notes
1. **Initialization**: At engine startup, `SetupPrefsParseTree()` is called to build the XML parser hierarchy
2. **Loading**: XML parser tree is used to deserialize preferences from file via `XML_DataBlockLoader`
3. **User Interaction**: When user opens preferences, `handle_preferences()` displays main dialog. Selecting a button (e.g., PLAYER) launches that category's dialog (e.g., `player_dialog()`).
4. **Dialog Pattern**: Each dialog collects changes via widgets, validates, and on "ACCEPT", updates global preference structures and calls `write_preferences()`; on "CANCEL", discards changes.
5. **Saving**: `write_preferences()` persists all global preference pointers to XML file (implementation not in this file).
6. **Crosshair Preview**: Uses `BinderSet` to sync UI changes to a preview widget in real-time; only committed if user accepts.

## External Dependencies
- **UI Framework**: `dialog`, `w_title`, `w_button`, `w_select`, `w_text_entry`, `w_toggle`, `w_slider`, `w_password_entry`, `w_enabling_toggle`, `w_color_picker`, `w_player_color`, `w_crosshair_display` (defined elsewhere; SDL-based)
- **Dialogs/Widgets**: `vertical_placer`, `horizontal_placer`, `table_placer`, `BinderSet`, `Bindable<T>` template (preference binding)
- **XML Parsing**: `XML_ElementParser`, `XML_DataBlock`, `ColorParser` (color element parsing)
- **File I/O**: `FileSpecifier`, `FileHandler.h`
- **Game Data**: Graphics/network/player/input/sound/environment preference structures (defined in respective headers, referenced globally)
- **Network**: `StarGameProtocol`, `RingGameProtocol` (provide parsers for network game protocol sub-elements)
- **Utilities**: `StringsEqual()`, `ReadInt16Value()`, `ReadBooleanValue()`, etc. (XML value parsing helpers, defined elsewhere)
- **Platform**: `GetUserName()` (Windows), `getlogin()` (Unix), `a1_c2pstr()` (string conversion)
