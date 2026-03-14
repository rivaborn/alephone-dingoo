# Source_Files/Network/ConnectPool.cpp

## File Purpose
Implements a non-blocking TCP connection pool for the Aleph One engine. Manages outbound connection establishment in worker threads, allowing DNS resolution and socket connection without blocking the main thread. Uses a fixed-size pool of reusable connection objects.

## Core Responsibilities
- Spawn and manage worker threads for individual TCP connection attempts
- Perform DNS resolution (hostname ΓåÆ IP) in worker threads via SDL_Net
- Track connection state (Connecting, Connected, ResolutionFailed, ConnectFailed)
- Allocate/deallocate connection slots from a fixed-size pool
- Defer cleanup of completed connections until next allocation request
- Destroy threads safely in destructors

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `NonblockingConnect::Status` | enum | Connection lifecycle state (Connecting, Connected, ResolutionFailed, ConnectFailed) |
| `ConnectPool::m_pool` | array of std::pair | Fixed 20-slot pool; pair holds (NonblockingConnect*, bool is_available) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `ConnectPool::m_instance` | ConnectPool* | static | Singleton instance for global access |

## Key Functions / Methods

### NonblockingConnect::NonblockingConnect(string, uint16)
- **Signature:** `NonblockingConnect(const std::string& address, uint16 port)`
- **Purpose:** Initialize a connection attempt to a hostname/port pair; launch background resolution and connection.
- **Inputs:** `address` (hostname), `port` (network port)
- **Outputs/Return:** Object state set to `Connecting`; worker thread created
- **Side effects:** Spawns SDL thread; calls `connect()` to initiate async work
- **Calls:** `connect()`
- **Notes:** Sets `m_ipSpecified = false` to signal that DNS resolution is needed

### NonblockingConnect::NonblockingConnect(IPaddress)
- **Signature:** `NonblockingConnect(const IPaddress& ip)`
- **Purpose:** Initialize a connection attempt to a known IP address; skip DNS resolution.
- **Inputs:** `ip` (resolved IP address)
- **Outputs/Return:** Object state set to `Connecting`; worker thread created
- **Side effects:** Spawns SDL thread
- **Calls:** `connect()`
- **Notes:** Sets `m_ipSpecified = true`; bypasses DNS lookup

### NonblockingConnect::Thread()
- **Signature:** `int Thread()`
- **Purpose:** Main worker thread entry point; performs DNS resolution (if needed) and socket connection.
- **Inputs:** None (uses member state)
- **Outputs/Return:** 0 if connected, 1 if DNS failed, 2 if connect failed; updates `m_status`
- **Side effects:** Calls `SDLNet_ResolveHost()` (blocks thread), calls `m_channel->connect()`, populates `m_channel` on success
- **Calls:** `SDLNet_ResolveHost()`, `CommunicationsChannel::connect()`
- **Notes:** Worker threads exit after this completes; main thread checks status via `done()` and `status()`

### ConnectPool::connect(string, uint16) / connect(IPaddress)
- **Signature:** `NonblockingConnect* connect(const std::string& address, uint16 port)` and `NonblockingConnect* connect(const IPaddress& ip)`
- **Purpose:** Allocate a free slot from the pool and initiate a new connection attempt.
- **Inputs:** Target address/port or IP
- **Outputs/Return:** Pointer to allocated `NonblockingConnect` object, or null if pool is full
- **Side effects:** Calls `fast_free()` to clean up completed connections; marks slot as in-use (second=false)
- **Calls:** `fast_free()`, `NonblockingConnect` constructor
- **Notes:** Returns null when all 20 slots are occupied; does not wait

### ConnectPool::fast_free()
- **Signature:** `void fast_free()`
- **Purpose:** Clean up finished connection objects in the pool.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Deletes completed `NonblockingConnect` objects; resets slot pointers to null
- **Calls:** `NonblockingConnect::done()`
- **Notes:** Only frees marked available slots (second=true); does not force cleanup of in-use objects

### ConnectPool::abandon(NonblockingConnect*)
- **Signature:** `void abandon(NonblockingConnect* nbc)`
- **Purpose:** Mark a connection as abandoned, freeing its slot for cleanup and reallocation.
- **Inputs:** Pointer to a `NonblockingConnect` in the pool
- **Outputs/Return:** None
- **Side effects:** Sets slot availability flag to true (second=true)
- **Calls:** None
- **Notes:** Does not immediately delete; deferred cleanup happens on next `connect()` call via `fast_free()`

## Control Flow Notes
Not inferable from this file, but likely pattern:
- **Init/Load:** `ConnectPool::instance()` creates singleton
- **Update:** Main thread calls `ConnectPool::connect()` to initiate connections; polls `NonblockingConnect::status()` / `done()` 
- **Release:** `ConnectPool::abandon()` when connection result is consumed or timeout occurs
- **Shutdown:** Pool destructor (currently no-op; cleanup commented out)

## External Dependencies
- **SDL:** `SDL_Thread`, `SDL_CreateThread()`, `SDL_WaitThread()`, `SDLNet_ResolveHost()`
- **Custom:** `CommunicationsChannel` (defined elsewhere), `IPaddress` type (from cseries.h)
- **Standard Library:** `std::string`, `std::auto_ptr`, `std::pair`
