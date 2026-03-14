# Source_Files/XML/XML_MakeRoot.cpp

## File Purpose

Constructs the root of the XML parser tree for Marathon/Aleph One configuration files. Declares global root parser objects and initializes the complete parser hierarchy by attaching subsystem parsers (text strings, interface, player, items, weapons, etc.) as children of the main "marathon" element.

## Core Responsibilities

- Declare and maintain `RootParser` (absolute root) and `MarathonParser` ("marathon" element)
- Build the complete XML parse tree hierarchy via `SetupParseTree()`
- Provide initialization entry point for XML configuration system
- Enable bulk reset of all MML (Marathon Markup Language) configuration values to defaults

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `XML_ElementParser` | class | Represents an XML element in the parse tree; used to parse and manage configuration hierarchies |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `RootParser` | `XML_ElementParser` | global | Absolute root element of the entire XML document tree |
| `MarathonParser` | `XML_ElementParser` | global | Root element for Marathon configuration ("marathon" tag); child of `RootParser` |

## Key Functions / Methods

### SetupParseTree
- **Signature:** `void SetupParseTree()`
- **Purpose:** Builds the complete XML parser hierarchy by registering all subsystem parsers as children of `MarathonParser`.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Modifies global `MarathonParser` and `RootParser` objects; registers ~30 child parsers
- **Calls (direct calls visible):**
  - `RootParser.AddChild(&MarathonParser)`
  - `MarathonParser.AddChild(TS_GetParser())` (text strings)
  - `MarathonParser.AddChild(Interface_GetParser())`
  - `MarathonParser.AddChild(PlayerName_GetParser())`
  - `MarathonParser.AddChild(Infravision_GetParser())`
  - `MarathonParser.AddChild(MotionSensor_GetParser())`
  - `MarathonParser.AddChild(OverheadMap_GetParser())`
  - `MarathonParser.AddChild(DynamicLimits_GetParser())`
  - `MarathonParser.AddChild(AnimatedTextures_GetParser())`
  - `MarathonParser.AddChild(Player_GetParser())`
  - `MarathonParser.AddChild(Items_GetParser())`
  - `MarathonParser.AddChild(ControlPanels_GetParser())`
  - `MarathonParser.AddChild(Liquids_GetParser())`
  - `MarathonParser.AddChild(Sounds_GetParser())`
  - `MarathonParser.AddChild(Platforms_GetParser())`
  - `MarathonParser.AddChild(Scenery_GetParser())`
  - `MarathonParser.AddChild(Faders_GetParser())`
  - `MarathonParser.AddChild(View_GetParser())`
  - `MarathonParser.AddChild(Landscapes_GetParser())`
  - `MarathonParser.AddChild(Weapons_GetParser())`
  - `MarathonParser.AddChild(OpenGL_GetParser())`
  - `MarathonParser.AddChild(Cheats_GetParser())`
  - `MarathonParser.AddChild(TextureLoading_GetParser())`
  - `MarathonParser.AddChild(Keyboard_GetParser())`
  - `MarathonParser.AddChild(DamageKicks_GetParser())`
  - `MarathonParser.AddChild(Logging_GetParser())`
  - `MarathonParser.AddChild(Scenario_GetParser())`
  - (conditional) `MarathonParser.AddChild(Theme_GetParser())` (SDL only)
  - (conditional) `MarathonParser.AddChild(SW_Texture_Extras_GetParser())` (non-Dingoo)
  - `MarathonParser.AddChild(Console_GetParser())`
  - `MarathonParser.AddChild(ExternalDefaultLevelScript_GetParser())`
- **Notes:** Unconditionally called at initialization; subsystem parsers are obtained via `*_GetParser()` functions defined in their respective modules. Conditional compilation (#ifdef SDL, #ifndef HAVE_DINGOO) controls platform-specific parsers.

### ResetAllMMLValues
- **Signature:** `void ResetAllMMLValues()`
- **Purpose:** Resets all MML-configured values to their hardcoded defaults by delegating to children of `MarathonParser`.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Modifies game state via child parser reset methods
- **Calls (direct calls visible):** `MarathonParser.ResetChildrenValues()`
- **Notes:** Provides a bulk reset mechanism for all subsystems that implement MML value resetting.

## Control Flow Notes

Initialization: `SetupParseTree()` must be called before any XML configuration file is parsed. This constructs the parser tree that `XML_ElementParser` will use to route parsed XML elements to appropriate subsystem handlers.

Reset: `ResetAllMMLValues()` is called when the user needs to revert MML customizations (e.g., cheats disabled, level reset) without restarting the engine.

## External Dependencies

- **Notable includes:**
  - `XML_ParseTreeRoot.h` (declares `RootParser`, `SetupParseTree()`, `ResetAllMMLValues()`)
  - Subsystem headers (interface.h, player.h, weapons.h, etc.) for subsystem-specific declarations
  - `XML_LevelScript.h` for level script parser

- **External symbols (defined elsewhere):**
  - `TS_GetParser()`, `Interface_GetParser()`, `PlayerName_GetParser()`, `Infravision_GetParser()`, `MotionSensor_GetParser()`, `OverheadMap_GetParser()`, `DynamicLimits_GetParser()`, `AnimatedTextures_GetParser()`, `Player_GetParser()`, `Items_GetParser()`, `ControlPanels_GetParser()`, `Liquids_GetParser()`, `Sounds_GetParser()`, `Platforms_GetParser()`, `Scenery_GetParser()`, `Faders_GetParser()`, `View_GetParser()`, `Landscapes_GetParser()`, `Weapons_GetParser()`, `OpenGL_GetParser()`, `Cheats_GetParser()`, `TextureLoading_GetParser()`, `Keyboard_GetParser()`, `DamageKicks_GetParser()`, `Logging_GetParser()`, `Scenario_GetParser()`, `Theme_GetParser()`, `SW_Texture_Extras_GetParser()`, `Console_GetParser()`, `ExternalDefaultLevelScript_GetParser()` ΓÇö all return `XML_ElementParser*` and are defined in their respective subsystem modules.
