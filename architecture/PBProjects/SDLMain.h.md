# PBProjects/SDLMain.h

## File Purpose
Header file defining the main application controller class for an SDL-based game on macOS. Declares the interface for SDLMain, which bridges Cocoa UI framework events (menu actions) to the underlying game engine logic.

## Core Responsibilities
- Define the SDLMain application controller interface
- Declare menu action handlers (preferences, game creation/loading, saving, help)
- Serve as the primary event dispatcher for user-initiated game actions in Cocoa

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| SDLMain | class | Main application controller; inherits from NSObject; handles Cocoa menu/UI events |

## Global / File-Static State
None.

## Key Functions / Methods

### prefsMenu:
- Signature: `- (IBAction)prefsMenu:(id)sender;`
- Purpose: Handle preferences menu selection
- Inputs: `sender` (typically the UI element that triggered the action)
- Outputs/Return: void (IBAction)
- Side effects: Not inferable from header; likely opens preferences dialog or applies settings
- Calls: Not visible in header (implementation elsewhere)
- Notes: IBAction indicates this is wired to a Cocoa interface element

### newGame:
- Signature: `- (IBAction)newGame:(id)sender;`
- Purpose: Initiate a new game
- Inputs: `sender`
- Outputs/Return: void (IBAction)
- Side effects: Not inferable from header; likely resets game state and begins new session
- Calls: Not visible in header
- Notes: Menu action

### openGame:, saveGame:, saveGameAs:, help:
- Similar IBAction signatures; likely handle game loading, saving, and help display respectively

## Control Flow Notes
This is a header-only interface declaration. At runtime, these IBAction methods respond to Cocoa menu/UI events, dispatching control to the underlying game logic. Execution flow from UI ΓåÆ SDLMain methods ΓåÆ game engine implementation not visible here.

## External Dependencies
- `<Cocoa/Cocoa.h>` ΓÇô macOS Cocoa framework (GUI, AppKit, Foundation)
- Implicit dependency: SDL library (referenced in comments; actual SDL calls in implementation)
