# Source_Files/Input/ISp_Support.h

## File Purpose
Header file declaring the public interface for InputSprocket (ISp) support in the Marathon/Aleph One game engine. Provides lifecycle management and control configuration for an older macOS input abstraction API.

## Core Responsibilities
- Initialize and shut down the InputSprocket subsystem
- Start and stop input event monitoring during gameplay
- Query input device state (keyboard vs. other controllers)
- Test and enumerate available InputSprocket input elements
- Configure Marathon-specific control bindings for discovered input devices

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### initialize_ISp
- Signature: `void initialize_ISp(void)`
- Purpose: Initialize the InputSprocket subsystem for use
- Inputs: None
- Outputs/Return: None
- Side effects: Sets up InputSprocket runtime state; likely allocates resources
- Calls: Not visible in this file
- Notes: Must be called before any other ISp function; counterpart is `ShutDown_ISp()`

### ShutDown_ISp
- Signature: `void ShutDown_ISp(void)`
- Purpose: Clean up and disable the InputSprocket subsystem
- Inputs: None
- Outputs/Return: None
- Side effects: Deallocates InputSprocket resources; disables input monitoring
- Calls: Not visible in this file
- Notes: Counterpart to `initialize_ISp()`

### Start_ISp
- Signature: `void Start_ISp(void)`
- Purpose: Enable input event monitoring (called during frame/gameplay initialization)
- Inputs: None
- Outputs/Return: None
- Side effects: Activates InputSprocket event pumping
- Calls: Not visible in this file
- Notes: Called after `initialize_ISp()`; disabled by `Stop_ISp()`

### Stop_ISp
- Signature: `void Stop_ISp(void)`
- Purpose: Disable input event monitoring (called during pause or shutdown)
- Inputs: None
- Outputs/Return: None
- Side effects: Halts InputSprocket event delivery
- Calls: Not visible in this file

### ISp_IsUsingKeyboard
- Signature: `bool ISp_IsUsingKeyboard(void)`
- Purpose: Query whether keyboard is the active input device
- Inputs: None
- Outputs/Return: Boolean flag indicating keyboard usage
- Side effects: None (read-only query)
- Calls: Not visible in this file
- Notes: Used to distinguish input device type at runtime

### InputSprocketTestElements
- Signature: `long InputSprocketTestElements(void)`
- Purpose: Test/enumerate available InputSprocket input elements
- Inputs: None
- Outputs/Return: Long integer (likely count of elements or test result code)
- Side effects: Possibly enumerates or probes connected input devices
- Calls: Not visible in this file

### ConfigureMarathonISpControls
- Signature: `void ConfigureMarathonISpControls(void)`
- Purpose: Bind Marathon-specific game controls to discovered InputSprocket elements
- Inputs: None
- Outputs/Return: None
- Side effects: Modifies InputSprocket element-to-action mappings
- Calls: Not visible in this file
- Notes: Must be called after initialization but before meaningful input can occur

## Control Flow Notes
Expected initialization sequence: `initialize_ISp()` ΓåÆ `ConfigureMarathonISpControls()` ΓåÆ `Start_ISp()` ... gameplay loop ... `Stop_ISp()` ΓåÆ `ShutDown_ISp()`. This file is the public interface layer; InputSprocket state machine is managed elsewhere.

## External Dependencies
- InputSprocket API (macOS-specific; implementation likely in a paired .c/.cpp file)
- Standard C library (`bool`, `void`, `long` types)
