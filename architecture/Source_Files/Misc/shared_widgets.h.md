# Source_Files/Misc/shared_widgets.h

## File Purpose
Cross-platform UI widget binding abstraction layer. Provides bidirectional adapters between preference storage formats (Pascal strings, C strings, booleans, bit flags, file paths) and a common `Bindable<T>` interface. Conditionally includes SDL or Carbon platform-specific widget implementations and defines chat history infrastructure for networked games.

## Core Responsibilities
- Adapt various preference storage types to `Bindable<T>` interface for data binding
- Support platform-specific file path representations (FSSpec on Mac, char* buffer elsewhere)
- Provide conditional platform selection (SDL vs. Carbon) at compile time
- Implement observer pattern for chat history notifications (networking only)
- Abstract chat widget operations (append, clear) from platform implementation

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| PStringPref | class | Bindable adapter for Pascal string (unsigned char*) with length |
| CStringPref | class | Bindable adapter for C string (char*) with length |
| BoolPref | class | Bindable adapter for boolean reference |
| BitPref | class | Bindable adapter for bit mask in uint16 with optional inversion |
| Int16Pref | class | Bindable adapter for int16_t to int conversion |
| FilePref | class | Platform-conditional: FSSpec (Mac) or char buffer (non-Mac) to FileSpecifier |
| ChatHistory | class | Container and observable for ColoredChatEntry items |
| ChatHistory::NotificationAdapter | class | Abstract observer for content changes |
| ColoredChatEntry | struct | Chat message: type, sender color, sender name, message text |
| ColorfulChatWidget | class | Observer adapter linking ChatHistory to platform-specific widget impl |

## Global / File-Static State
None.

## Key Functions / Methods

### PStringPref / CStringPref bind_export / bind_import
- Signature: `virtual std::string bind_export()` / `virtual void bind_import(std::string s)`
- Purpose: Convert between preference storage and std::string
- Inputs: s (std::string for import)
- Outputs/Return: std::string (export)
- Side effects: None (export); modifies preference buffer (import, respects m_length)
- Calls: `pstring_to_string()`, `copy_string_to_pstring()`, `copy_string_to_cstring()` (cseries)
- Notes: CStringPref uses direct constructor; PStringPref via helper functions

### BoolPref bind_export / bind_import
- Signature: `virtual bool bind_export()` / `virtual void bind_import(bool value)`
- Purpose: Direct boolean reference binding
- Outputs/Return: m_pref (bool reference)
- Side effects: None (export); assigns value (import)
- Notes: Trivial passthrough; no conversion

### BitPref bind_export
- Signature: `virtual bool bind_export()`
- Purpose: Extract masked bits as boolean with optional inversion
- Outputs/Return: `(m_invert ? !(m_pref & m_mask) : (m_pref & m_mask))`
- Notes: Non-zero result treated as true; inversion applied before export

### BitPref bind_import
- Signature: `virtual void bind_import(bool value)`
- Purpose: Set/clear bits in mask; apply inversion before update
- Outputs/Return: None
- Side effects: `m_pref |= m_mask` (if value), `m_pref &= ~m_mask` (if !value)
- Notes: Inversion logic: `(m_invert ? !value : value)` determines operation

### Int16Pref bind_export / bind_import
- Signature: `virtual int bind_export()` / `virtual void bind_import(int value)`
- Purpose: Type adapter between int16_t and int
- Outputs/Return: m_pref cast to int
- Notes: Simple passthrough with implicit narrowing/widening

### FilePref bind_export / bind_import (platform variants)
- **Mac** ΓÇö Signature: `virtual FileSpecifier bind_export()` / `virtual void bind_import(FileSpecifier value)`
  - Converts: FSSpec Γåö FileSpecifier via `SetSpec()` / `GetSpec()`
- **Non-Mac** ΓÇö Converts: char[256] path buffer Γåö FileSpecifier via constructor / `GetPath()` + `strncpy()`
- Side effects: None (export); writes to buffer, truncated to 255 chars (import non-Mac)

### ChatHistory::append
- Signature: `void append(const ColoredChatEntry& e)`
- Purpose: Add entry to history and notify observer
- Inputs: e (ColoredChatEntry)
- Outputs/Return: None
- Side effects: Appends to m_history; calls `observer->contentAdded(e)` if registered
- Calls: `NotificationAdapter::contentAdded()`
- Notes: Only compiled if `!DISABLE_NETWORKING`

### ChatHistory::clear
- Signature: `void clear()`
- Purpose: Empty history and notify observer
- Side effects: Clears m_history; calls `observer->contentCleared()` if set
- Notes: Complement to append; enables chat UI reset

### ChatHistory::setObserver
- Signature: `void setObserver(NotificationAdapter* adapter)`
- Purpose: Register/unregister observer
- Inputs: adapter (pointer, may be NULL)
- Side effects: Updates m_notificationAdapter
- Notes: Allows NULL to unregister

### ColorfulChatWidget::attachHistory
- Signature: `void attachHistory(ChatHistory* history)`
- Purpose: Subscribe widget to history notifications
- Inputs: history (ChatHistory instance)
- Side effects: Stores m_history; registers self as observer via `history->setObserver(this)`
- Notes: Bidirectional setup; assumes constructor initializes m_notificationAdapter = NULL

### ColorfulChatWidget::contentAdded / contentCleared
- Signature: `virtual void contentAdded(const ColoredChatEntry& e)` / `virtual void contentCleared()`
- Purpose: Implement NotificationAdapter callbacks
- Inputs: e (entry added)
- Side effects: Forwards to `m_componentWidget->Append()` (contentAdded) / `->Clear()` (contentCleared)
- Notes: Platform-specific impl delegated to ColorfulChatWidgetImpl (SDL or Carbon)

## Control Flow Notes
Not tied to engine frame loop. Preference classes support bidirectional binding, typically invoked during dialog initialization/synchronization. Chat history is event-driven: append/clear calls trigger observer notifications asynchronously during gameplay (networking mode only). No frame/update dependencies.

## External Dependencies
- **cseries.h** ΓÇö Core types, string conversion functions (`pstring_to_string`, `copy_string_to_pstring`, `copy_string_to_cstring`)
- **sdl_widgets.h** or **carbon_widgets.h** ΓÇö Platform-specific widget implementations (conditional include)
- **binders.h** ΓÇö `Bindable<T>` template interface
- **config.h** ΓÇö `DISABLE_NETWORKING` compile flag
- **FileHandler.h** ΓÇö `FileSpecifier` cross-platform file path abstraction (external definition)
- `ColorfulChatWidgetImpl` ΓÇö Platform implementation (SDL/Carbon, defined elsewhere)
