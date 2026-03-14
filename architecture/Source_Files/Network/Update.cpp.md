# Source_Files/Network/Update.cpp

## File Purpose
Implements online update checking for the Aleph One game engine. Spawns a background thread to connect to the SourceForge update server, fetch platform-specific version information via HTTP, and maintain the update availability status without blocking the main game loop.

## Core Responsibilities
- Manage singleton lifecycle for the Update checker
- Spawn and manage a background SDL thread for non-blocking network I/O
- Establish TCP connections to the update server (marathon.sourceforge.net)
- Construct and send HTTP GET requests for platform-specific update metadata
- Parse HTTP responses to extract version information (`A1_DATE_VERSION`, `A1_DISPLAY_VERSION`)
- Maintain and expose update status (checking, available, failed, none)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `Update` | class | Singleton managing update checking; holds status, version strings, thread handle |
| `Status` | enum | Four states: `CheckingForUpdate`, `UpdateCheckFailed`, `UpdateAvailable`, `NoUpdateAvailable` |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `Update::m_instance` | `Update*` | static | Singleton instance pointer |

## Key Functions / Methods

### Update (constructor)
- **Signature:** `Update()`
- **Purpose:** Initialize the singleton and launch the first update check
- **Inputs:** (none)
- **Outputs/Return:** (void)
- **Side effects:** Spawns background thread; sets `m_status` to `CheckingForUpdate`
- **Calls:** `StartUpdateCheck()`
- **Notes:** No thread-safety synchronization; assumes single construction

### ~Update (destructor)
- **Signature:** `~Update()`
- **Purpose:** Clean up background thread on shutdown
- **Inputs:** (none)
- **Outputs/Return:** (void)
- **Side effects:** Blocks until background thread completes (`SDL_WaitThread`)
- **Calls:** `SDL_WaitThread()`
- **Notes:** Defensive null-check on `m_thread` before wait

### StartUpdateCheck
- **Signature:** `void StartUpdateCheck()`
- **Purpose:** Initiate a new update check; cancels any in-flight check and spawns a fresh thread
- **Inputs:** (none)
- **Outputs/Return:** (void)
- **Side effects:** Clears `m_new_date_version` and `m_new_display_version`; sets `m_status` to `CheckingForUpdate` or `UpdateCheckFailed`
- **Calls:** `SDL_CreateThread()`, `SDL_WaitThread()`
- **Notes:** Early-exit if already checking; waits for previous thread before spawning new one

### Thread
- **Signature:** `int Update::Thread()`
- **Purpose:** Execute the actual update check: resolve hostname, connect via TCP, send HTTP request, parse response
- **Inputs:** (none; accesses member variables)
- **Outputs/Return:** int status code (0 on success, 1ΓÇô5 on various failures)
- **Side effects:** 
  - Resolves DNS for "marathon.sourceforge.net"
  - Opens TCP socket (port 80)
  - Sends raw HTTP GET request
  - Receives up to 8192 bytes of response
  - Parses response with `strtok()`; populates `m_new_date_version` and `m_new_display_version`
  - Sets `m_status` based on version comparison
- **Calls:** `SDLNet_ResolveHost()`, `SDLNet_TCP_Open()`, `SDLNet_TCP_Send()`, `SDLNet_TCP_Recv()`, `strtok()`, `sprintf()`, `strncmp()`, `strlen()`, string `.assign()`, `.compare()`, `.size()` / `.clear()`
- **Notes:** 
  - Hard-coded server address; no dynamic endpoint
  - Fixed 1024-byte request buffer with unbounded `sprintf()` (potential overflow)
  - Fixed 8192-byte reply buffer
  - TODO comments: does not handle HTTP `Location:` redirects or chunked transfer encoding
  - Uses string comparison (`A1_DATE_VERSION.compare()`) to determine if newer version is available
  - No retry logic; fails immediately on any socket error

### update_thread (static trampoline)
- **Signature:** `static int update_thread(void *p)`
- **Purpose:** SDL thread entry point; casts opaque pointer and delegates to member `Thread()`
- **Inputs:** `p` ΓÇô opaque pointer to `Update` instance
- **Outputs/Return:** return value from `Thread()`
- **Side effects:** (none)
- **Calls:** `Thread()`
- **Notes:** Required because SDL thread API expects C-style function pointer

## Control Flow Notes
- **Initialization:** Constructor calls `StartUpdateCheck()`, which spawns a background thread immediately
- **Async/Frame-Independent:** Network I/O happens in a separate thread; main game loop remains responsive
- **Status Polling:** Caller periodically checks `GetStatus()` to learn when results are ready
- **Shutdown:** Destructor blocks until thread completes

## External Dependencies
- **SDL_net:** `SDLNet_ResolveHost()`, `SDLNet_TCP_Open()`, `SDLNet_TCP_Send()`, `SDLNet_TCP_Recv()`
- **SDL_thread:** `SDL_CreateThread()`, `SDL_WaitThread()`
- **C standard library:** `strtok()`, `strncmp()`, `strlen()`, `sprintf()`
- **C++ standard library:** `std::string` (assign, compare, size, clear)
- **alephversion.h:** `A1_UPDATE_PLATFORM`, `A1_DATE_VERSION` (version constants)
- **boost/tokenizer.hpp:** included but not used in this file

**Security/Robustness Notes:**
- `sprintf()` without bounds checking (buffer overflow risk)
- No HTTPS/TLS support; HTTP only
- Hard-coded server address not configurable
- No timeout on socket operations (could hang indefinitely)
- No certificate validation or man-in-the-middle protection
