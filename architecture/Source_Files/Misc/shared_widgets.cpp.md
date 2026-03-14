# Source_Files/Misc/shared_widgets.cpp

## File Purpose
Implements shared chat UI widgets for the Aleph One game engine, bridging platform-independent chat history management with platform-specific widget rendering. Provides observer pattern integration between chat data and UI display layers for both SDL and Carbon (macOS) backends.

## Core Responsibilities
- Maintains colored chat message history with notification callbacks
- Synchronizes chat history state with widget display via observer pattern
- Manages lifecycle of chat widget and its attached history source
- Notifies registered observers when chat content is added or cleared
- Populates widget display when history source is attached mid-session

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `ChatHistory` | class | Manages a vector of `ColoredChatEntry` items; provides append/clear operations and observer notification |
| `ChatHistory::NotificationAdapter` | abstract class | Observer interface for chat content changes (contentAdded, contentCleared) |
| `ColorfulChatWidget` | class | Concrete observer implementing `NotificationAdapter`; displays chat history in platform-specific widget |
| `ColoredChatEntry` | class/struct | Represents a single chat message with color metadata (defined elsewhere) |
| `ColorfulChatWidgetImpl` | class | Platform-specific widget implementation (defined in sdl_widgets.h or carbon_widgets.h) |

## Global / File-Static State
None.

## Key Functions / Methods

### ChatHistory::append
- **Signature:** `void append(const ColoredChatEntry& e)`
- **Purpose:** Add a new chat entry to history and notify observer
- **Inputs:** `e` ΓÇö colored chat entry to append
- **Outputs/Return:** None
- **Side effects:** Modifies `m_history` vector; calls `m_notificationAdapter->contentAdded(e)` if adapter is set
- **Calls:** `vector::push_back()`, `NotificationAdapter::contentAdded()`
- **Notes:** Safe null-check on adapter before calling

### ChatHistory::clear
- **Signature:** `void clear()`
- **Purpose:** Remove all chat entries and notify observer
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Clears `m_history` vector; calls `m_notificationAdapter->contentCleared()` if adapter is set
- **Calls:** `vector::clear()`, `NotificationAdapter::contentCleared()`
- **Notes:** Safe null-check on adapter

### ColorfulChatWidget::attachHistory
- **Signature:** `void attachHistory(ChatHistory* history)`
- **Purpose:** Attach a history source, detach previous one, and synchronize display with all current entries
- **Inputs:** `history` ΓÇö pointer to ChatHistory to observe (null allowed)
- **Outputs/Return:** None
- **Side effects:** Clears widget display; populates widget with all entries from new history; updates observer registration on old/new history objects
- **Calls:** `ChatHistory::setObserver()`, `ColorfulChatWidgetImpl::Clear()`, `ColorfulChatWidgetImpl::Append()`, `ChatHistory::getHistory()`
- **Notes:** Iterates entire history vector on attach; handles null history gracefully

### ColorfulChatWidget::contentAdded
- **Signature:** `void contentAdded(const ColoredChatEntry& e)` [virtual]
- **Purpose:** Append a single entry to the widget display (callback from observed history)
- **Inputs:** `e` ΓÇö colored chat entry to display
- **Outputs/Return:** None
- **Side effects:** Calls widget implementation's `Append()`
- **Calls:** `ColorfulChatWidgetImpl::Append()`

### ColorfulChatWidget::~ColorfulChatWidget
- **Purpose:** Clean up widget and detach observer
- **Side effects:** Calls `ChatHistory::setObserver(0)` if history is attached; deletes `m_componentWidget`
- **Notes:** Virtual destructor; safe null-check on `m_history`

## Control Flow Notes
Part of the UI/dialog initialization and runtime layers. `attachHistory()` is called during widget setup to bind a history source. Subsequently, `ChatHistory::append/clear` trigger callbacks that update the display. Fits into frame-loop or event-driven architecture where chat messages arrive and must be reflected immediately in the UI.

## External Dependencies
- **Includes:** `cseries.h`, `preferences.h`, `player.h`, `shared_widgets.h`, `<vector>`, `<algorithm>`
- **Used but defined elsewhere:** `ColoredChatEntry`, `ColorfulChatWidgetImpl` (platform-specific in sdl_widgets.h or carbon_widgets.h)
- **Standard library:** `std::vector`
- **Conditional:** Entire implementation wrapped in `#if !defined(DISABLE_NETWORKING)` (networking disabled on Dingoo platform)
