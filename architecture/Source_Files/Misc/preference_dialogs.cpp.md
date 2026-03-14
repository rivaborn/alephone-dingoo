# Source_Files/Misc/preference_dialogs.cpp

## File Purpose
Implements OpenGL graphics preference dialogs for the Aleph One game engine. Provides a concrete SDL-based UI for configuring texture quality, filtering, antialiasing, and other graphics settings, with automatic preference persistence via a binder/converter pattern.

## Core Responsibilities
- Define preference converter classes that translate between UI controls and internal preference formats (bit shifting, scaling, filtering)
- Implement abstract `OpenGLDialog` base class lifecycle and preference binding orchestration
- Implement concrete `SdlOpenGLDialog` class: UI construction, tab layout, widget hierarchy, and dialog execution
- Manage bidirectional preference synchronization: UI ΓåÆ preferences on accept, preferences ΓåÆ UI on open
- Provide factory method for platform-independent dialog instantiation

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `TexQualityPref` | class / Bindable | Converts int16 texture size limits (e.g., 128, 256) to discrete quality levels via bit shifting |
| `ColourPref` | class / Bindable | Pass-through converter for RGBColor preference values |
| `FilterPref` | class / Bindable | Maps UI popup indices (0-based) to odd-numbered filter constants (1, 3, 5, ...) |
| `TimesTwoPref` | class / Bindable | Converts int16 values to/from doubled integers (e.g., 1 Γåö 2, 2 Γåö 4) |
| `AnisotropyPref` | class / Bindable | Converts float anisotropy level to/from integer exponent (powers of 2) via bit shifting |
| `OpenGLDialog` | class | Abstract base; declares UI widget members and preference binding orchestration |
| `SdlOpenGLDialog` | class | Concrete SDL implementation; constructs dialog UI, owns placer and dialog objects |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `filter_labels` | `const char*[4]` | static | Labels for texture filtering modes ("Linear", "Bilinear", "Trilinear", NULL) |

## Key Functions / Methods

### OpenGLDialog::~OpenGLDialog()
- **Signature:** `virtual ~OpenGLDialog()`
- **Purpose:** Clean up all managed widget pointers allocated by concrete subclass
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Deallocates ~17 widget member pointers and arrays of texture quality/resolution/depth widgets
- **Calls:** operator `delete` (standard deallocation)
- **Notes:** Assumes all widget pointers are either initialized or null; safe for partially initialized dialogs

### OpenGLDialog::OpenGLPrefsByRunning()
- **Signature:** `void OpenGLPrefsByRunning()`
- **Purpose:** Display dialog, load/save preferences via bidirectional binding, and persist on accept
- **Inputs:** None (reads from `graphics_preferences` global, modifies via binder callbacks)
- **Outputs/Return:** None
- **Side effects:** 
  - Calls `Run()` (blocks until user closes dialog)
  - Calls `write_preferences()` if accepted
  - Modifies `graphics_preferences->OGL_Configure` fields if dialog accepted
- **Calls:**
  - `BinderSet::insert<bool/int/RGBColor>()` (creates preference bindings)
  - `BinderSet::migrate_all_second_to_first()` (load prefs ΓåÆ UI)
  - `Run()` (abstract, implemented in `SdlOpenGLDialog`)
  - `BinderSet::migrate_all_first_to_second()` (UI ΓåÆ save prefs, only if accepted)
  - `write_preferences()`
- **Notes:** 
  - Creates ~25 Bindable objects (local stack-allocated) to convert between preference formats and UI controls
  - All BitPref converters map to flags in `graphics_preferences->OGL_Configure.Flags`
  - BinderSet cleanup is automatic (destructor deletes all Binder objects)

### SdlOpenGLDialog::SdlOpenGLDialog()
- **Signature:** `SdlOpenGLDialog()`
- **Purpose:** Construct the dialog UI hierarchy: tabs, tables, toggles, sliders, popups, and buttons
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** 
  - Allocates dynamic UI objects (placer, tables, widgets)
  - Populates `m_dialog` with widget placer
  - Initializes all member widget pointers (from `OpenGLDialog` base)
  - Initializes `m_tabs` (tab_placer pointer)
- **Calls:**
  - Widget constructors: `w_title`, `w_toggle`, `w_select_popup`, `w_slider`, `w_select`, `w_button`, `w_label`, `w_static_text`
  - Placer methods: `dual_add()`, `add()`, `add_row()`, `dual_add_row()`, `set_widget_placer()`, `col_flags()`, `col_min_width()`
  - Widget methods: `label()`, `set_labels()`, `associate_label()`, `set_callback()`
  - Wrapper constructors: `ButtonWidget`, `ToggleWidget`, `PopupSelectorWidget`, `SliderSelectorWidget`, `SelectSelectorWidget`
  - `get_theme_space()` (layout constant)
- **Notes:**
  - Organizes UI into two tabs: "GENERAL" (quality, effects, antialiasing) and "ADVANCED" (filters, built-in texture size/depth)
  - Uses 2-column and 3-column table layouts
  - Texture quality/resolution/depth arrays are indexed by `OGL_NUMBER_OF_TEXTURE_TYPES` (4 types)
  - Four texture types: walls, landscapes, sprites/inhabitants, weapons in hand
  - Void color widgets initialized to null (0) ΓÇö not shown in SDL dialog

### SdlOpenGLDialog::Run()
- **Signature:** `virtual bool Run()`
- **Purpose:** Execute dialog event loop and return user's choice
- **Inputs:** None
- **Outputs/Return:** `bool` ΓÇö true if user clicked ACCEPT, false if CANCEL/closed
- **Side effects:** Blocks until dialog closes; calls callbacks set via `set_callback()`
- **Calls:** `m_dialog.run()` (returns 0 on OK, -1 on cancel)
- **Notes:** Returns `m_dialog.run() == 0`

### SdlOpenGLDialog::Stop(bool result)
- **Signature:** `virtual void Stop(bool result)`
- **Purpose:** Close the dialog with the given result
- **Inputs:** `result` ΓÇö true to accept changes, false to cancel
- **Outputs/Return:** None
- **Side effects:** Quits `m_dialog` with exit code (0 for accept, -1 for cancel)
- **Calls:** `m_dialog.quit(int)`, invoked by OK/CANCEL button callbacks
- **Notes:** Connected via `boost::bind` in `OpenGLPrefsByRunning()`

### SdlOpenGLDialog::choose_generic_tab(void *arg)
- **Signature:** `static void choose_generic_tab(void *arg)`
- **Purpose:** Switch to the GENERAL tab (currently unused; code is commented out in UI construction)
- **Inputs:** `arg` ΓÇö void pointer to `SdlOpenGLDialog` instance
- **Outputs/Return:** None
- **Side effects:** Calls `m_tabs->choose_tab(0)` and redraws dialog
- **Calls:** `m_tabs->choose_tab()`, `m_dialog->draw()`
- **Notes:** Static callback function; not wired into current UI

### SdlOpenGLDialog::choose_advanced_tab(void *arg)
- **Signature:** `static void choose_advanced_tab(void *arg)`
- **Purpose:** Switch to the ADVANCED tab (currently unused; code is commented out in UI construction)
- **Inputs:** `arg` ΓÇö void pointer to `SdlOpenGLDialog` instance
- **Outputs/Return:** None
- **Side effects:** Calls `m_tabs->choose_tab(1)` and redraws dialog
- **Calls:** `m_tabs->choose_tab()`, `m_dialog->draw()`
- **Notes:** Static callback function; not wired into current UI

### OpenGLDialog::Create()
- **Signature:** `static std::auto_ptr<OpenGLDialog> Create()`
- **Purpose:** Factory method for platform-independent dialog instantiation
- **Inputs:** None
- **Outputs/Return:** `auto_ptr<OpenGLDialog>` wrapping a new `SdlOpenGLDialog` instance
- **Side effects:** Allocates a new dialog object on the heap
- **Calls:** `SdlOpenGLDialog` constructor
- **Notes:** Concrete type is chosen at link-time; enables iOS/macOS variants to override

## Control Flow Notes
**Initialization:** Dialog is instantiated via `Create()` ΓåÆ `SdlOpenGLDialog` constructor builds entire UI hierarchy, storing widget pointers in inherited member variables.

**Preferences Loop:** `OpenGLPrefsByRunning()` is the main entry point:
1. Set OK/CANCEL button callbacks to call `Stop(true/false)`
2. Create preference converters and pair them with widgets via `BinderSet`
3. Load prefs ΓåÆ UI via `migrate_all_second_to_first()` (reads from `graphics_preferences`)
4. Block in `Run()` until user closes dialog
5. If accepted, migrate UI ΓåÆ prefs via `migrate_all_first_to_second()` and call `write_preferences()`

**Render:** UI framework handles rendering (not visible in this file).

## External Dependencies
- **Includes:**
  - `preference_dialogs.h` ΓÇö abstract interface and widget member declarations
  - `preferences.h` ΓÇö global `graphics_preferences` pointer, `write_preferences()`
  - `binders.h` ΓÇö `Bindable<T>`, `Binder<T>`, `BinderSet` template classes
  - `OGL_Setup.h` ΓÇö `OGL_ConfigureData`, `OGL_Flag_*` constants, texture type enums, `RGBColor`

- **External symbols (defined elsewhere):**
  - `graphics_preferences` ΓÇö global preferences struct
  - `write_preferences()` ΓÇö persists prefs to disk
  - Widget classes: `w_title`, `w_toggle`, `w_select_popup`, `w_slider`, `w_select`, `w_button`, `w_label`, `w_static_text`, `dialog`
  - Wrapper/selector classes: `ButtonWidget`, `ToggleWidget`, `PopupSelectorWidget`, `SliderSelectorWidget`, `SelectSelectorWidget`
  - Placer classes: `vertical_placer`, `horizontal_placer`, `table_placer`, `tab_placer`
  - UI utilities: `get_theme_space()`
  - `boost::bind` ΓÇö callback binding

- **Notes on dependencies:** Widget and placer classes suggest a custom UI toolkit (SDL-based, likely); no standard library widgets are used. The binder pattern decouples preference conversion logic from UI framework.
