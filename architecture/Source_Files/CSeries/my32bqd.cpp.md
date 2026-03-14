# Source_Files/CSeries/my32bqd.cpp

## File Purpose
Provides a portability abstraction layer for Macintosh QuickDraw graphics operations, wrapping low-level GWorld and menu bar APIs. Abstracts differences between Classic Mac and Carbon/modern Mac implementations.

## Core Responsibilities
- Wrap QuickDraw GWorld (graphics world) creation, management, and disposal
- Provide pixel locking/unlocking and pixel map access abstractions
- Implement menu bar visibility control with platform-specific backends
- Stub out low-level color palette operations
- Provide initialization hook for graphics subsystem

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| eMenuBarState | enum | Tracks menu bar state (UNKNOWN/HIDDEN/SHOWING) for Carbon platforms |
| GWorldPtr | typedef | Pointer to graphics world (from Mac API) |
| GDHandle | typedef | Handle to graphics device (from Mac API) |
| PixMapHandle | typedef | Handle to pixel map (from Mac API) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| savedgray | RgnHandle | static | Saved gray region before hiding menu bar (Classic Mac only) |
| savedmbh | short | static | Saved menu bar height before hiding (Classic Mac only) |
| mbarstate | eMenuBarState | static | Current menu bar state tracking (Carbon only) |

## Key Functions / Methods

### myHideMenuBar / myShowMenuBar
- **Signature:** `void myHideMenuBar(GDHandle ignoredDev)` / `void myShowMenuBar(void)`
- **Purpose:** Toggle menu bar visibility with platform-aware backends
- **Inputs:** (ignoredDev unused for Carbon)
- **Outputs/Return:** void
- **Side effects:** Modifies system gray region and menu bar height (Classic); calls native Carbon APIs and state tracking
- **Calls:** GetGrayRgn, NewRgn, CopyRgn, GetDeviceList, GetNextDevice, TestDeviceAttribute, RectRgn, UnionRgn, DisposeRgn, LMGetMBarHeight, LMSetMBarHeight (Classic); HideMenuBar, ShowMenuBar, dprintf (Carbon)
- **Notes:** Two completely separate implementations: Classic Mac uses low-level region/low-memory management; Carbon uses native APIs with state machine to prevent redundant calls. Carbon version includes debug warnings for state violations.

### LowLevelSetEntries
- **Signature:** `void LowLevelSetEntries(short start, short count0, ColorSpec *specs)`
- **Purpose:** Set color table entries (palette)
- **Inputs:** start (first entry), count0 (number of entries), specs (color specifications)
- **Outputs/Return:** void
- **Side effects:** Would modify device color table if implemented
- **Calls:** assert (always fails)
- **Notes:** Currently disabled with `assert(0)` ΓÇö original implementation commented out due to unavailable device control under Carbon. Device drivers no longer accessible from user code.

### Trivial Wrapper Functions
Thin single-line passthroughs to native Mac API:
- `myGetGWorld` / `mySetGWorld` ΓÇö Get/set active graphics world
- `myNewGWorld` / `myUpdateGWorld` ΓÇö Create/update graphics worlds
- `myDisposeGWorld` ΓÇö Destroy graphics world
- `myLockPixels` / `myUnlockPixels` ΓÇö Lock/unlock pixel buffer access
- `myGetGWorldPixMap` ΓÇö Get pixel map from graphics world
- `myGetPixBaseAddr` ΓÇö Get raw pixel buffer pointer

### initialize_my_32bqd
- **Signature:** `void initialize_my_32bqd(void)`
- **Purpose:** Initialization hook
- **Outputs/Return:** void (empty)
- **Notes:** Stub; no initialization currently required

## Control Flow Notes
- Called during engine initialization/shutdown for menu bar management
- Called during render setup/teardown for graphics world operations
- Pixel access functions called when reading/writing framebuffer data
- Two completely different code paths for Carbon vs. Classic Mac, selected at compile time

## External Dependencies
- **Includes:** `Carbon/Carbon.h` (if `EXPLICIT_CARBON_HEADER` defined); `my32bqd.h` (header)
- **Conditional includes (Carbon only):** `csalerts.h` (for `dprintf`), `csstrings.h`
- **External symbols:** GWorld/GDevice/PixMap functions from Mac QuickDraw API; region and low-memory management functions (Classic only)
- **Defined elsewhere:** All wrapped Mac API functions (`GetGWorld`, `SetGWorld`, `NewGWorld`, `HideMenuBar`, etc.)
