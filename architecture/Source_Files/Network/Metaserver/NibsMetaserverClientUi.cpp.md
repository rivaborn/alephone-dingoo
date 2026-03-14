# Source_Files/Network/Metaserver/NibsMetaserverClientUi.cpp

## File Purpose
Carbon/macOS-specific UI implementation for the metaserver client. Loads Interface Builder NIB files and instantiates a modal dialog with widgets for players, games, chat, and associated controls. Provides the concrete factory implementation of `MetaserverClientUi`.

## Core Responsibilities
- Load and manage Carbon NIB resources ("Metaserver Client")
- Create and initialize UI control widgets (list views, text entry, buttons)
- Implement the `MetaserverClientUi` abstract interface for macOS
- Establish a periodic polling timer to pump metaserver client events at ~30 Hz
- Manage modal dialog lifecycle (Run/Stop)
- Bridge abstract metaserver logic with Carbon framework UI

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `NibsMetaserverClientUi` | class | Concrete implementation of `MetaserverClientUi` for Carbon/macOS |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `PollingInterval` | const double | global | Timer interval for metaserver event polling (1.0/30.0 seconds) |

## Key Functions / Methods

### NibsMetaserverClientUi (constructor)
- Signature: `NibsMetaserverClientUi()`
- Purpose: Initialize metaserver client UI by loading NIB and creating all widgets
- Inputs: None
- Outputs/Return: None
- Side effects: Loads "Metaserver Client" NIB from resources; allocates widget objects (ListWidget, ButtonWidget, EditTextWidget, HistoricTextboxWidget); initializes Modal_Dialog
- Calls: `AutoNibReference`, `AutoNibWindow`, `Modal_Dialog` constructors; `GetCtrlFromWindow()` (├ù5); widget constructors
- Notes: Uses RAII patternΓÇöauto-managing pointers ensure resource cleanup; widget initialization delegates to helper classes defined in headers

### Run (virtual)
- Signature: `virtual void Run()`
- Purpose: Start the UI event loop and begin processing metaserver events
- Inputs: None
- Outputs/Return: None (returns when dialog closes)
- Side effects: Creates event loop timer; runs modal dialog; processes Carbon events until Stop() is called
- Calls: `AutoTimer` constructor; `Modal_Dialog::Run()`
- Notes: Timer fires at `PollingInterval`; stack-allocated timer ensures cleanup on return

### Stop (virtual)
- Signature: `virtual void Stop()`
- Purpose: Terminate the UI event loop and close the dialog
- Inputs: None
- Outputs/Return: None
- Side effects: Closes modal dialog window
- Calls: `Modal_Dialog::Stop(false)`
- Notes: Passes `false` result to Stop()

### MetaserverClientUi_Poller (static, pascal)
- Signature: `static pascal void MetaserverClientUi_Poller(EventLoopTimerRef, void*)`
- Purpose: Timer callback to pump pending metaserver client events
- Inputs: `EventLoopTimerRef` (unused), `void*` user data (unused)
- Outputs/Return: None
- Side effects: Calls `MetaserverClient::pumpAll()` to process queued messages
- Calls: `MetaserverClient::pumpAll()`
- Notes: Invoked periodically by Carbon event loop; uses Pascal calling convention (macOS-specific)

### MetaserverClientUi::Create (static factory)
- Signature: `static std::auto_ptr<MetaserverClientUi> MetaserverClientUi::Create()`
- Purpose: Abstract factory that returns concrete metaserver client UI implementation
- Inputs: None
- Outputs/Return: `std::auto_ptr<MetaserverClientUi>` owning a new `NibsMetaserverClientUi`
- Side effects: Allocates heap object
- Calls: `NibsMetaserverClientUi` constructor
- Notes: Actual type created determined at link-time; enables platform-specific UI implementations

## Control Flow Notes
UI is event-driven. On `Run()`: NIB loads, dialog enters event loop, polling timer fires ~30 times per second to call `MetaserverClientUi_Poller()`, which pumps metaserver events. User interactions and incoming messages trigger widget callbacks (inherited from base class). `Stop()` or user window-close triggers dialog exit.

## External Dependencies
- `metaserver_dialogs.h`: `MetaserverClientUi` base class, `GlobalMetaserverChatNotificationAdapter`
- `NibsUiHelpers.h`: `AutoNibReference`, `AutoNibWindow`, `Modal_Dialog`, `AutoTimer`, `GetCtrlFromWindow()`
- `shared_widgets.h`: `ListWidget`, `ButtonWidget`, `EditTextWidget`, `HistoricTextboxWidget`, `TextboxWidget`
- `boost/function.hpp`, `boost/bind.hpp`, `boost/static_assert.hpp`: Utility templates (minimal use in this file)
- Carbon framework: `EventLoopTimerRef`, `pascal` calling convention (defined elsewhere)
- `MetaserverClient` (singleton, defined elsewhere)
- `GameListMessage::GameListEntry`, `MetaserverPlayerInfo` types (defined elsewhere)
