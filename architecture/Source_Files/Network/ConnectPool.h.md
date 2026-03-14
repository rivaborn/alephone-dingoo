# Source_Files/Network/ConnectPool.h

## File Purpose
Provides a thread-based connection pool for initiating non-blocking outbound TCP connections. Manages up to 20 concurrent connection attempts via `NonblockingConnect` objects and reuses idle connections through the singleton `ConnectPool`.

## Core Responsibilities
- **NonblockingConnect**: Manage a single asynchronous TCP connection attempt
  - Spawn and monitor a background SDL thread for non-blocking connection
  - Track connection lifecycle (Connecting ΓåÆ Connected/Failed)
  - Support DNS resolution (string address) and direct IP connection
  - Wrap successful connections in `CommunicationsChannel` for messaging
- **ConnectPool**: Manage a pool of reusable connection objects
  - Singleton pattern for application-wide access
  - Allocate and recycle `NonblockingConnect` instances
  - Track in-use vs. available slots (bool flag in pair)
  - Clean up abandoned connections

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `NonblockingConnect::Status` | enum | Lifecycle states: `Connecting`, `Connected`, `ResolutionFailed`, `ConnectFailed` |
| `std::auto_ptr<CommunicationsChannel>` | typedef | Owned reference to communication channel; released when connection is claimed |
| `std::pair<NonblockingConnect*, bool>` | type | Pool entry: pointer + "in-use" flag (false = available, true = in-use) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `ConnectPool::m_instance` | `ConnectPool*` | static | Singleton instance; lazily initialized in `instance()` |
| `m_thread` | `SDL_Thread*` | instance | Background thread executing connection attempt for each `NonblockingConnect` |

## Key Functions / Methods

### NonblockingConnect(const std::string& address, uint16 port)
- **Signature:** Constructor
- **Purpose:** Initiate a connection to a named host/IP and port.
- **Inputs:** DNS-resolvable address string, port number.
- **Outputs/Return:** None (constructor).
- **Side effects:** Launches SDL thread; begins async DNS/connect sequence; updates `m_status`.
- **Calls:** `connect()`, `connect_thread()`.
- **Notes:** Address is stored for later DNS resolution in background thread.

### NonblockingConnect(const IPaddress& ip)
- **Signature:** Constructor
- **Purpose:** Initiate a connection to a pre-resolved IP address.
- **Inputs:** Already-resolved `IPaddress` (from SDL_net).
- **Outputs/Return:** None (constructor).
- **Side effects:** Launches SDL thread; begins async connect sequence; avoids DNS.
- **Calls:** `connect()`, `connect_thread()`.
- **Notes:** Skips DNS resolution step (`m_ipSpecified = true`).

### Status status()
- **Signature:** `Status status()`
- **Purpose:** Query current connection state.
- **Inputs:** None.
- **Outputs/Return:** One of `{Connecting, Connected, ResolutionFailed, ConnectFailed}`.
- **Side effects:** None.
- **Notes:** Non-blocking read of `m_status` set by background thread.

### bool done()
- **Signature:** `bool done()`
- **Purpose:** Convenience check: has connection attempt finished (success or failure)?
- **Inputs:** None.
- **Outputs/Return:** `true` if `m_status != Connecting`.
- **Side effects:** None.

### const IPaddress& address()
- **Signature:** `const IPaddress& address()`
- **Purpose:** Retrieve resolved/connected IP address.
- **Inputs:** None.
- **Outputs/Return:** Reference to `m_ip`.
- **Side effects:** None.
- **Notes:** Asserts `m_status` is neither `Connecting` nor `ResolutionFailed`; safe only after `done() == true` and status is not a failure.

### CommunicationsChannel* release()
- **Signature:** `CommunicationsChannel* release()`
- **Purpose:** Extract and surrender ownership of the connected `CommunicationsChannel`.
- **Inputs:** None.
- **Outputs/Return:** Raw pointer to `CommunicationsChannel`; caller must delete.
- **Side effects:** Clears internal `auto_ptr` (ownership transferred).
- **Notes:** Asserts `m_status == Connected`; must be called exactly once per successful connection.

### ConnectPool* ConnectPool::instance()
- **Signature:** `static ConnectPool* instance()`
- **Purpose:** Obtain or create the singleton pool instance.
- **Inputs:** None.
- **Outputs/Return:** Pointer to global `ConnectPool`.
- **Side effects:** First call allocates `m_instance` on heap.
- **Notes:** Not thread-safe in initialization; assumes single-threaded startup or external synchronization.

### NonblockingConnect* ConnectPool::connect(const std::string& address, uint16 port)
- **Signature:** `NonblockingConnect* connect(const std::string&, uint16)`
- **Purpose:** Request a connection from the pool by hostname/IP and port.
- **Inputs:** Address string, port.
- **Outputs/Return:** Pointer to `NonblockingConnect`; caller does not own (pool owns).
- **Side effects:** Reuses available slot or creates new entry if space available.
- **Calls:** `NonblockingConnect` constructor.

### NonblockingConnect* ConnectPool::connect(const IPaddress& ip)
- **Signature:** `NonblockingConnect* connect(const IPaddress&)`
- **Purpose:** Request a connection from the pool by pre-resolved IP.
- **Inputs:** `IPaddress` struct.
- **Outputs/Return:** Pointer to `NonblockingConnect`; caller does not own.
- **Side effects:** Reuses or creates connection attempt.

### void ConnectPool::abandon(NonblockingConnect*)
- **Signature:** `void abandon(NonblockingConnect*)`
- **Purpose:** Return a connection to the pool for reuse.
- **Inputs:** Pointer previously returned by `connect()`.
- **Outputs/Return:** None.
- **Side effects:** Marks pool slot as available (bool = `false`).

## Control Flow Notes
1. **Initialization:** `ConnectPool::instance()` creates singleton on first call.
2. **Outbound Connection:** Caller invokes `pool->connect(address, port)` ΓåÆ pool creates or reuses `NonblockingConnect` ΓåÆ background thread begins async connection.
3. **Polling:** Caller polls `conn->status()` or `conn->done()` in game loop until connection succeeds/fails.
4. **Message Exchange:** On success, caller invokes `conn->release()` to obtain `CommunicationsChannel` and uses it for I/O.
5. **Return to Pool:** After use, caller invokes `pool->abandon(conn)` to mark the slot available for future reuse.
6. **Cleanup:** `ConnectPool::~ConnectPool()` and `fast_free()` clean up outstanding connections.

## External Dependencies
- `"config.h"` ΓÇö Provides `DISABLE_NETWORKING` guard and `HAVE_SDL_NET`.
- `"cseries.h"` ΓÇö Core types and macros (uint16 typedef, etc.).
- `"CommunicationsChannel.h"` ΓÇö Defines `CommunicationsChannel`, `IPaddress` (via `<SDL_net.h>`).
- `<string>`, `<memory>` ΓÇö STL containers and smart pointers.
- `<SDL_thread.h>` ΓÇö SDL threading API (`SDL_Thread`).
- **Defined elsewhere:** `IPaddress`, `TCPsocket` (from SDL_net), `CommunicationsChannel`.
