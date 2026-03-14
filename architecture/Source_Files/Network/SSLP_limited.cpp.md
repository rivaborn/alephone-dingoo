# Source_Files/Network/SSLP_limited.cpp

## File Purpose
Implementation of the Simple Service Location Protocol (SSLP) for network service discovery. Enables applications to locate remote services (FIND), advertise local services (HAVE), and notify when services become unavailable (LOST). Single-threaded design requires periodic calls to SSLP_Pump() from the main thread to process network activity.

## Core Responsibilities
- Manage UDP socket for SSLP packet transmission and reception
- Maintain linked list of discovered service instances with timeout tracking
- Handle three message types: FIND (requests), HAVE (responses), LOST (notifications)
- Track active behaviors via state flags (LOCATING, RESPONDING, HINTING)
- Serialize/deserialize packets with architecture-independent packing
- Invoke user-provided callbacks when services are found, lost, or renamed
- Broadcast FIND requests and listen for responses at regular 5-second intervals

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| SSLPint_FoundInstance | struct | Linked-list node holding discovered service instance, discovery timestamp, and next pointer |
| SSLP_Packet | struct (Protocol.h) | Network packet format with magic, version, message type, service port, type, and name |
| SSLP_ServiceInstance | struct (API.h) | Public API structure with service type, name, and network address |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| sBehaviorsDesired | int | static | Bit flags tracking active SSLP behaviors (LOCATING, RESPONDING, HINTING) |
| sSocketDescriptor | UDPsocket | static | Shared UDP socket for all SSLP communication |
| sReceivingPacket | UDPpacket* | static | Pre-allocated buffer for incoming packets |
| sFindPacket | UDPpacket* | static | Pre-allocated FIND request packet template for broadcasts |
| sFoundCallback | callback | static | Invoked when service instance discovered |
| sLostCallback | callback | static | Invoked when service instance becomes unavailable |
| sNameChangedCallback | callback | static | Invoked when service name changes at same host:port |
| sFoundInstances | SSLPint_FoundInstance* | static | Linked list of discovered service instances |
| sHintPacket | UDPpacket* | static | Pre-allocated HAVE packet for unicast hinting |
| sResponsePacket | UDPpacket* | static | Pre-allocated HAVE packet for responding to FIND |

## Key Functions / Methods

### SSLPint_FoundAnInstance
- **Signature:** `static struct SSLP_ServiceInstance* SSLPint_FoundAnInstance(struct SSLP_ServiceInstance* inInstance)`
- **Purpose:** Register newly discovered service or update existing instance's name/timestamp
- **Inputs:** Service instance with type, name, host, port
- **Outputs/Return:** Pointer to new instance if previously unknown; NULL if already tracked
- **Side effects:** Allocates memory for new instances, updates sFoundInstances list, invokes sNameChangedCallback on name change, updates timestamp
- **Calls:** malloc(), strncmp(), strncpy(), SDL_GetTicks(), callbacks
- **Notes:** Matches by host+port only. Returned pointer remains valid until sFoundInstances modified.

### SSLPint_RemoveTimedOutInstances
- **Signature:** `void SSLPint_RemoveTimedOutInstances()`
- **Purpose:** Purge instances not heard from within 20000ms timeout
- **Side effects:** Removes expired entries, frees memory, invokes sLostCallback (if LOCATING active)
- **Calls:** SDL_GetTicks(), free()
- **Notes:** Walks entire list. Only invokes callbacks if LOCATING flag set.

### SSLPint_LostAnInstance
- **Signature:** `static struct SSLP_ServiceInstance* SSLPint_LostAnInstance(struct SSLP_ServiceInstance* inInstance)`
- **Purpose:** Remove tracked service instance by host+port match
- **Outputs/Return:** Pointer to removed instance (NULL if not found)
- **Side effects:** Removes from sFoundInstances, frees wrapper only (not instance itself)
- **Notes:** Caller must free() returned pointer if non-NULL.

### SSLPint_ReceivedPacket
- **Signature:** `static void SSLPint_ReceivedPacket()`
- **Purpose:** Unpack and dispatch received SSLP packet (FIND/HAVE/LOST)
- **Side effects:** Validates magic/version, processes message type, invokes callbacks, modifies sFoundInstances, may send response packets
- **Calls:** UnpackPacket(), SDL_SwapBE32/16(), SSLPint_FoundAnInstance(), SSLPint_LostAnInstance(), SDLNet_UDP_Send()
- **Notes:** Early returns prevent further processing on validation failures.

### SSLPint_Enter
- **Signature:** `static bool SSLPint_Enter()`
- **Purpose:** Initialize socket, broadcast capability, and receive buffer on first use
- **Outputs/Return:** true on success; false if socket/allocation fails
- **Side effects:** Opens UDP socket on SSLP_PORT (fallback to port 0), enables broadcast, allocates sReceivingPacket
- **Calls:** SDLNet_UDP_Open(), SDLNetx_EnableBroadcast(), SDLNet_AllocPacket()
- **Notes:** Broadcast failure is non-fatal. Uses port 0 if SSLP_PORT unavailable.

### SSLPint_Exit
- **Signature:** `static int SSLPint_Exit()`
- **Purpose:** Release socket and packet buffers
- **Side effects:** Closes socket, frees packet, nullifies both
- **Calls:** SDLNet_UDP_Close(), SDLNet_FreePacket()
- **Notes:** Asserts sBehaviorsDesired == SSLPINT_NONE.

### SSLP_Locate_Service_Instances
- **Signature:** `void SSLP_Locate_Service_Instances(const char* inServiceType, SSLP_Service_Instance_Status_Changed_Callback inFoundCallback, ...)`
- **Purpose:** Start searching for service instances; register status-change callbacks
- **Side effects:** Initializes SSLP if inactive, allocates sFindPacket, sets callbacks, sets LOCATING flag
- **Calls:** SSLPint_Enter(), SDLNet_AllocPacket(), PackPacket()
- **Notes:** Asserts LOCATING not already active.

### SSLP_Stop_Locating_Service_Instances
- **Signature:** `void SSLP_Stop_Locating_Service_Instances(const char* inServiceType)`
- **Purpose:** Cease service discovery; clear callbacks and instances
- **Side effects:** Clears LOCATING flag, nullifies callbacks, frees sFindPacket, flushes found instances, calls SSLPint_Exit() if no other behaviors
- **Notes:** inServiceType parameter ignored (implementation limitation).

### SSLP_Allow_Service_Discovery
- **Signature:** `void SSLP_Allow_Service_Discovery(const struct SSLP_ServiceInstance* inServiceInstance)`
- **Purpose:** Advertise local service; respond to FIND requests with HAVE
- **Side effects:** Initializes SSLP if needed, allocates sResponsePacket, broadcasts HAVE once, sets RESPONDING flag
- **Calls:** SSLPint_Enter(), SDLNet_AllocPacket(), PackPacket(), SDLNetx_UDP_Broadcast()
- **Notes:** Asserts RESPONDING/HINTING not already active.

### SSLP_Hint_Service_Discovery
- **Signature:** `void SSLP_Hint_Service_Discovery(const struct SSLP_ServiceInstance* inServiceInstance, const IPaddress* inAddress)`
- **Purpose:** Periodically unicast HAVE to specific host (cross-subnet discovery)
- **Side effects:** Enables RESPONDING if needed, allocates/populates sHintPacket, sets HINTING flag
- **Calls:** SSLP_Allow_Service_Discovery(), SDLNet_AllocPacket(), PackPacket()
- **Notes:** Uses SSLP_PORT if inAddress->port is 0.

### SSLP_Disallow_Service_Discovery
- **Signature:** `void SSLP_Disallow_Service_Discovery(const struct SSLP_ServiceInstance* inInstance)`
- **Purpose:** Stop advertising service; send LOST notifications
- **Side effects:** If hinting, sends unicast LOST; broadcasts LOST on SSLP_PORT; frees packets; clears flags; calls SSLPint_Exit() if no behaviors remain
- **Calls:** UnpackPacket(), PackPacket(), SDLNet_UDP_Send(), SDLNetx_UDP_Broadcast(), SDLNet_FreePacket()
- **Notes:** inInstance ignored. Asserts RESPONDING active.

### SSLP_Pump
- **Signature:** `void SSLP_Pump()`
- **Purpose:** Main processing function; must be called periodically from main thread
- **Side effects:** Broadcasts FIND every ~5s (if LOCATING), removes timed-out instances, unicasts HINT, receives and processes all pending packets
- **Calls:** SDL_GetTicks(), SDLNetx_UDP_Broadcast(), SSLPint_RemoveTimedOutInstances(), SDLNet_UDP_Recv(), SSLPint_ReceivedPacket()
- **Notes:** Uses static throttle variable. Drains all pending packets in single call. No-op if no behaviors active.

## Control Flow Notes
**Lazy Init:** SSLP_Enter() called on first API function call (Locate/Allow/Hint). Opens socket and allocates buffers.

**Main Loop:** SSLP_Pump() drives all activityΓÇömust be called every 1ΓÇô5 seconds (avoid >5s gaps). Throttles broadcasts to ~5s intervals, processes all pending packets, ages out stale instances.

**Shutdown:** SSLP_Stop_Locating_Service_Instances() and SSLP_Disallow_Service_Discovery() decrement behavior flags. When sBehaviorsDesired reaches 0, SSLP_Exit() releases resources.

**Callbacks:** Invoked synchronously in SSLP_Pump()'s thread context (main thread in single-threaded design).

## External Dependencies
- **SDL.h, SDL_endian.h:** SDL type definitions (Uint32, Uint16, SDL_GetTicks, SDL_SwapBE32/16)
- **SDL_net.h:** UDP socket/packet APIs (SDLNet_UDP_Open/Close, Alloc/FreePacket, Recv/Send, UDPsocket, UDPpacket, IPaddress)
- **SDL_netx.h:** Broadcast extensions (SDLNetx_EnableBroadcast, SDLNetx_UDP_Broadcast)
- **SSLP_API.h, SSLP_Protocol.h:** Protocol definitions (constants, SSLP_Packet, SSLP_ServiceInstance, callback typedef)
- **Logging.h:** Diagnostic output (logContext, logTrace, logAnomaly, logNote)
- **Standard C:** assert, string (strncmp, strncpy, memcpy, memset), stdlib (malloc, free)
