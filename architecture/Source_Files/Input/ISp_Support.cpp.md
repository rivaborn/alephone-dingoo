# Source_Files/Input/ISp_Support.cpp

## File Purpose
Implements InputSprocket (ISp) device support for the Marathon game engine on macOS. Maps hardware input (keyboards, mice, joysticks) to game actions through virtual input elements. Handles both global networked actions and local machine-only events (quit, pause, settings).

## Core Responsibilities
- Initialize/shutdown InputSprocket framework and virtual input elements
- Define and register 44 input actions: 21 global (networked), 23 local, and 4 analog axes
- Poll input devices each frame and convert device state to action flags
- Enforce refractory periods on local-event buttons to prevent rapid repeat triggering
- Support configuration UI and device activation/deactivation
- Handle Carbon API compatibility (disable ISp under Carbon)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| Icon_* enums (1000ΓÇô1203) | enum constants | Symbolic identifiers for input actions |
| Button_FirstGlobal, Button_LastGlobal, etc. | enum constants | Index ranges for action categories |
| gNeeds | struct array | ISpNeed[HowManyNeeds]; specs for all 44 virtual input elements |
| gVirtualElements | struct array | ISpElementReference[HowManyNeeds]; runtime refs to created elements |
| gElementActions | struct array | long[HowManyNeeds]; maps button indices to action flag bits |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| canDoISp | bool | static | Tracks whether ISp is available (weak-linked) |
| active | bool | static | Tracks whether ISp polling is currently active |
| LocalButtonTimes | int[23] | static | Refractory timers for local-event buttons |
| gVirtualList | ISpElementListReference | static | Handle to created virtual element list |
| gVirtualElements | ISpElementReference[44] | static | Runtime references to virtual elements |
| gElementActions | long[44] | static | Maps button index ΓåÆ action flag |

## Key Functions / Methods

### initialize_ISp
- **Signature:** `void initialize_ISp(void)`
- **Purpose:** Initialize InputSprocket and register all virtual input elements at startup.
- **Inputs:** None; reads global `input_preferences`.
- **Outputs/Return:** None; sets `canDoISp`, `active`, populates `gVirtualElements`.
- **Side effects:** Calls ISp API (ISpStartup, ISpElement_NewVirtualFromNeeds, ISpInit, ISpSuspend); modifies global device state.
- **Calls (direct):** ISpStartup, ISpDevices_ActivateClass, ISpDevices_DeactivateClass, ISpElement_NewVirtualFromNeeds, ISpElementList_New, ISpElementList_AddElements, ISpInit, ISpSuspend; validates with vassert.
- **Notes:** Disabled on Carbon target. Weak-links ISp and returns early if unavailable. Creates 44 virtual elements (21 global, 23 local, 4 axes) from gNeeds array.

### InputSprocketTestElements
- **Signature:** `long InputSprocketTestElements(void)`
- **Purpose:** Poll all input devices and accumulate action flags for this frame; post local events to queue.
- **Inputs:** None; reads device state via ISp API.
- **Outputs/Return:** long action_flags; combined bits of _moving_forward, _left_trigger_state, etc.
- **Side effects:** Calls ISpTickle (updates ISp state); decrements LocalButtonTimes; calls PostLocalEvent for local actions; modifies global `flags` and `LocalButtonTimes`.
- **Calls (direct):** ISpTickle, ISpElement_GetSimpleState (global/local buttons), ISpElement_GetComplexState (4 analog axes), mask_in_absolute_positioning_information, PostLocalEvent.
- **Notes:** Returns 0 if inactive or unavailable. Global buttons (0ΓÇô20) are accumulated into flags; local buttons (21ΓÇô43) check refractory timer and post to LocalEventFlags. Axes (look yaw/pitch, move, weapon) are scaled (pitch ├ù2) and masked into flags or trigger weapon cycle.

### Start_ISp / Stop_ISp
- **Signature:** `void Start_ISp(void)` / `void Stop_ISp(void)`
- **Purpose:** Resume/suspend InputSprocket polling without destroying device state.
- **Inputs/Outputs:** None.
- **Side effects:** Calls ISpResume/ISpSuspend; toggles `active` flag.
- **Calls:** ISpDevices_ActivateClass/DeactivateClass (conditional), ISpResume/ISpSuspend.

### ShutDown_ISp
- **Signature:** `void ShutDown_ISp(void)`
- **Purpose:** Clean up and shut down InputSprocket at engine shutdown.
- **Side effects:** Disposes virtual elements, deactivates all device classes, calls ISpShutdown.
- **Calls:** ISpStop, ISpElement_DisposeVirtual, ISpDevices_DeactivateClass, ISpShutdown.

### ConfigureMarathonISpControls
- **Signature:** `void ConfigureMarathonISpControls(void)`
- **Purpose:** Open native InputSprocket configuration/binding dialog.
- **Inputs/Outputs:** None.
- **Side effects:** Resumes ISp, opens config UI, then suspends ISp again.
- **Calls:** ISpResume, ISpConfigure(nil), ISpSuspend.

### ISp_IsUsingKeyboard
- **Signature:** `bool ISp_IsUsingKeyboard(void)`
- **Purpose:** Query whether keyboard is the active input device.
- **Return:** true if ISp active and input_preferencesΓåÆinput_device == _input_sprocket_only.
- **Notes:** Trivial inline-eligible logic; returns false if ISp unavailable or inactive.

## Control Flow Notes
**Initialization flow:** Game startup ΓåÆ initialize_ISp() ΓåÆ create virtual elements ΓåÆ ISpSuspend (idle).  
**Per-frame input:** Engine loop ΓåÆ InputSprocketTestElements() polls devices ΓåÆ returns action_flags for physics update + posts LocalEvent flags to shell.  
**Shutdown:** Engine exit ΓåÆ ShutDown_ISp() disposes elements.  

InputSprocket runs in suspended state most of the time; Start_ISp/Stop_ISp toggle ISpResume/ISpSuspend without destroying device bindings.

## External Dependencies
- **Framework includes (macOS):** `InputSprocket.h`, `CursorDevices.h`, `Traps.h` (weak-linked; disabled on Carbon)
- **Project includes:** `world.h`, `map.h`, `player.h`, `preferences.h`, `LocalEvents.h`, `ISp_Support.h`, `macintosh_cseries.h`, `math.h`
- **Defined elsewhere:** `input_preferences` (global struct), `mask_in_absolute_positioning_information()`, `PostLocalEvent()`, `temporary` (formatting buffer for csprintf), action flag enums (_left_trigger_state, _moving_forward, etc.)
