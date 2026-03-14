# Source_Files/CSeries/my32bqd.h

## File Purpose
Provides a C compatibility/abstraction layer for 32-bit QuickDraw graphics operations on classic and Carbon macOS. Encapsulates GWorld (offscreen graphics buffer) lifecycle and related graphics context management to abstract platform-specific graphics APIs.

## Core Responsibilities
- Initialize the 32-bit QuickDraw subsystem
- Manage graphics world (GWorld) creation, updating, and disposal
- Get/set the current graphics drawing context and device
- Lock/unlock pixel buffers for direct memory access
- Query graphics world pixel maps and base addresses
- Manage menu bar visibility
- Configure color palette entries

## Key Types / Data Structures
None directly defined here; all types are imported from Carbon.h or QDOffscreen.h.

| Name (used) | Kind | Purpose |
|---|---|---|
| `GWorldPtr` | opaque pointer | Handle to an offscreen graphics world |
| `GDHandle` | opaque pointer | Reference to a graphics device |
| `Rect` | struct (defined elsewhere) | Rectangle coordinates for GWorld bounds |
| `CTabHandle` | opaque pointer | Color table for pixel mapping |
| `PixMapHandle` | opaque pointer | Handle to pixel map data structure |
| `ColorSpec` | struct (defined elsewhere) | Individual color table entry |

## Global / File-Static State
None.

## Key Functions / Methods

### initialize_my_32bqd
- Signature: `void initialize_my_32bqd(void)`
- Purpose: One-time initialization of the 32-bit QuickDraw compatibility layer.
- Inputs: None.
- Outputs/Return: None.
- Side effects: Likely initializes internal state, function pointers, or platform-specific graphics subsystem.
- Calls: Not visible.
- Notes: Must be called before any other functions in this module.

### myGetGWorld / mySetGWorld
- Signature: `void myGetGWorld(GWorldPtr *gw, GDHandle *dev)` / `void mySetGWorld(GWorldPtr gw, GDHandle dev)`
- Purpose: Query or change the current graphics context (graphics world + device).
- Inputs: For `mySetGWorld`: target graphics world and device.
- Outputs/Return: For `myGetGWorld`: output parameters receive current context.
- Side effects: Changes which offscreen buffer receives subsequent drawing commands.
- Calls: Not visible.
- Notes: Likely thin wrappers around QuickDraw `GetGWorld`/`SetGWorld`.

### myNewGWorld / myUpdateGWorld
- Signature: `OSErr myNewGWorld(GWorldPtr *gw, short depth, Rect *bounds, CTabHandle clut, GDHandle dev, unsigned long flags)` / similar for `myUpdateGWorld`
- Purpose: Create or resize/reconfigure an offscreen graphics world.
- Inputs: Desired pixel depth, bounds rectangle, color table, target device, configuration flags.
- Outputs/Return: `OSErr` status; GWorld pointer via output parameter.
- Side effects: Allocates/deallocates graphics memory.
- Calls: Not visible.
- Notes: `myUpdateGWorld` may modify the GWorld in-place or recreate it.

### myDisposeGWorld
- Signature: `void myDisposeGWorld(GWorldPtr gw)`
- Purpose: Deallocate an offscreen graphics world and free associated memory.
- Inputs: Graphics world to destroy.
- Outputs/Return: None.
- Side effects: Frees graphics memory.
- Calls: Not visible.
- Notes: Should not be called on a locked GWorld.

### myLockPixels / myUnlockPixels
- Signature: `bool myLockPixels(GWorldPtr gw)` / `void myUnlockPixels(GWorldPtr gw)`
- Purpose: Acquire or release exclusive access to pixel buffer memory.
- Inputs: Target graphics world.
- Outputs/Return: `myLockPixels` returns success; `myUnlockPixels` returns nothing.
- Side effects: Pins pixel memory in place; may disable QuickDraw drawing on unlocked buffers.
- Calls: Not visible.
- Notes: Locks must be paired; base address valid only while locked.

### myGetGWorldPixMap / myGetPixBaseAddr
- Signature: `PixMapHandle myGetGWorldPixMap(GWorldPtr gw)` / `Ptr myGetPixBaseAddr(GWorldPtr gw)`
- Purpose: Access the underlying pixel map and raw pixel buffer address.
- Inputs: Graphics world.
- Outputs/Return: Pixel map handle or base memory pointer.
- Side effects: None (read-only queries).
- Calls: Not visible.
- Notes: `myGetPixBaseAddr` requires pixels to be locked first.

### myHideMenuBar / myShowMenuBar
- Signature: `void myHideMenuBar(GDHandle dev)` / `void myShowMenuBar(void)`
- Purpose: Toggle menu bar visibility (fullscreen/windowed mode control).
- Inputs: `myHideMenuBar` takes target device; `myShowMenuBar` takes none.
- Outputs/Return: None.
- Side effects: Shows/hides system menu bar.
- Calls: Not visible.
- Notes: Asymmetric signatures suggest `myShowMenuBar` operates globally.

### LowLevelSetEntries
- Signature: `void LowLevelSetEntries(short start, short count0, ColorSpec *specs)`
- Purpose: Update color palette entries at low level.
- Inputs: Starting palette index, count of entries to update, array of color specifications.
- Outputs/Return: None.
- Side effects: Modifies color table in use.
- Calls: Not visible.
- Notes: Trivial helper for color palette manipulation.

## Control Flow Notes
This is a header-only interface; control flow is in the paired implementation file. Typical usage:
1. Call `initialize_my_32bqd()` during engine startup.
2. Create GWorlds with `myNewGWorld()` during scene/level initialization.
3. Use `mySetGWorld()` to switch drawing context.
4. Lock pixels, render, unlock pixels.
5. Dispose GWorlds during shutdown.

The module abstracts the transition from classic QuickDraw to Carbon-era APIs, likely handling version branching internally.

## External Dependencies
- `Carbon.h` or `QDOffscreen.h` ΓÇö macOS Carbon framework; provides `GWorldPtr`, `GDHandle`, `Rect`, `CTabHandle`, `PixMapHandle`, `ColorSpec`, `OSErr`.
- Conditional include suggests fallback compatibility paths for different macOS versions.
