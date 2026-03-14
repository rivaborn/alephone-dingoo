# Source_Files/Misc/preference_dialogs.h

## File Purpose
Defines an abstract factory for creating platform-specific OpenGL graphics preference dialogs in the Aleph One game engine. Manages UI widgets for configuring OpenGL rendering settings such as texture quality, filtering, anti-aliasing, and lighting effects.

## Core Responsibilities
- Abstract factory pattern (`Create()`) to instantiate platform-specific dialog implementations at link-time (Carbon or SDL)
- Encapsulates OpenGL graphics preference widgets (toggles, selectors, color pickers) for UI presentation
- Manages user interaction through OK/Cancel buttons
- Organizes configuration options across texture types (walls, landscapes, inhabitants, weapons-in-hand) and model quality
- Provides virtual interface for platform-specific dialog behavior (`Run`, `Stop`)

## Key Types / Data Structures
None (composed of widget types defined in dependency headers).

## Global / File-Static State
None.

## Key Functions / Methods

### Create
- Signature: `static std::auto_ptr<OpenGLDialog> Create()`
- Purpose: Abstract factory method that creates and returns a platform-specific OpenGLDialog implementation
- Inputs: None
- Outputs/Return: `std::auto_ptr<OpenGLDialog>` ΓÇö owning pointer to concrete subclass instance
- Side effects: Allocates heap memory for new dialog
- Calls: Concrete factory (not visible; determined at link-time)
- Notes: Uses `std::auto_ptr` for RAII; actual implementation (Carbon or SDL) chosen at link-time based on platform

### OpenGLPrefsByRunning
- Signature: `void OpenGLPrefsByRunning()`
- Purpose: Public entry point; displays and runs the OpenGL preferences dialog modally
- Inputs: None
- Outputs/Return: void
- Side effects: Shows dialog UI; user interactions update widget state; calls `Run()` and `Stop()`
- Calls: Platform-specific `Run()`, then `Stop(bool)`
- Notes: Hides platform-specific implementation details behind clean public interface

### Run (pure virtual)
- Signature: `virtual bool Run() = 0`
- Purpose: Platform-specific modal loop; processes user input and widget interactions
- Inputs: None
- Outputs/Return: bool (success indicator)
- Side effects: Modal dialog window; widget state changes reflect user input
- Calls: (Defined in subclass)
- Notes: Must be overridden by Carbon or SDL implementations

### Stop (pure virtual)
- Signature: `virtual void Stop(bool result) = 0`
- Purpose: Platform-specific cleanup; applies or discards preference changes based on result
- Inputs: `bool result` ΓÇö true if OK clicked (apply), false if Cancel clicked (discard)
- Outputs/Return: void
- Side effects: Closes dialog; updates OpenGL configuration if result==true
- Calls: (Defined in subclass)
- Notes: Called after `Run()` completes; result determines whether to persist widget state

## Control Flow Notes
Typical lifecycle:
1. `Create()` instantiates platform-specific subclass
2. `OpenGLPrefsByRunning()` displays modal preferences dialog
3. User interacts with widgets (toggles for Z-buffer/fog/effects, selectors for texture quality/filtering/resolution, color picker for void)
4. User clicks OK (m_okWidget) or Cancel (m_cancelWidget) button
5. Platform-specific `Stop(result)` closes dialog and conditionally commits preference changes
6. Dialog deallocated via `std::auto_ptr`

Fits into engine preferences/settings system; called from menus or initialization to configure OpenGL rendering at runtime.

## External Dependencies
- `shared_widgets.h` ΓÇö provides widget base classes: `ButtonWidget`, `ToggleWidget`, `SelectorWidget`, `SelectSelectorWidget`, `ColourPickerWidget`
- `OGL_Setup.h` ΓÇö provides `OGL_NUMBER_OF_TEXTURE_TYPES` constant (4: walls, landscapes, inhabitants, weapons-in-hand) and OpenGL configuration enums/flags
- Standard C++ library: `<memory>` (for `std::auto_ptr`)
