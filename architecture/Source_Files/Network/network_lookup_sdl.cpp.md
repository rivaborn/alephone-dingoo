# Source_Files/Network/network_lookup_sdl.cpp

## File Purpose
Implements network service discovery and player registration for the Aleph One game engine using SDL and SSLP (Simple Service Location Protocol). Handles locating remote player services on the network and publishing this machine's player service for discovery by others.

## Core Responsibilities
- Start/stop service discovery via SSLP callbacks
- Convert game engine's Pascal strings to SSLP-compatible C string format
- Construct and publish local player service instances
- Manage service registration lifecycle (register/unregister)
- Support service discovery hints for cross-broadcast-domain connectivity
- Provide stub functions for entity lookup operations (not yet implemented)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| SSLP_ServiceInstance | struct (from SSLP_API.h) | Encodes service type, name, and network address for discovery |
| OSErr | typedef (from cseries.h) | Error code type (0 = noErr) |
| IPaddress | typedef (from SDL_net) | Network address (host + port pair) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| sNameRegistered | bool | static | Tracks whether local player service is currently published |
| sLookupInProgress | bool | static | Tracks whether service discovery is active |

## Key Functions / Methods

### NetLookup_BuildTypeString
- **Signature:** `void NetLookup_BuildTypeString(char* outDest, const unsigned char* inType, short inVersion)`
- **Purpose:** Convert game's Pascal-string type + version into SSLP-compatible C-string format
- **Inputs:** Pascal string (inType, length byte + chars), version number (inVersion)
- **Outputs/Return:** Writes constructed string to outDest buffer
- **Side effects:** Truncates type to fit SSLP_MAX_TYPE_LENGTH constraints; overwrites outDest
- **Calls:** memcpy, sprintf, strlen
- **Notes:** Leaves room for length byte, null terminator, and longest short int string representation; pattern is `\p%s%d`

### NetLookupOpen_SSLP
- **Signature:** `OSErr NetLookupOpen_SSLP(const unsigned char *type, short version, SSLP_Service_Instance_Status_Changed_Callback foundInstance, SSLP_Service_Instance_Status_Changed_Callback lostInstance, SSLP_Service_Instance_Status_Changed_Callback nameChanged)`
- **Purpose:** Begin service discovery for a given service type
- **Inputs:** Service type (Pascal string), version, three callback function pointers
- **Outputs/Return:** OSErr (always 0/noErr)
- **Side effects:** Enables SSLP discovery, sets sLookupInProgress = true; callbacks invoked asynchronously by SSLP
- **Calls:** NetLookup_BuildTypeString, SSLP_Locate_Service_Instances
- **Notes:** One discovery attempt at a time; callbacks may fire in different threads

### NetRegisterName
- **Signature:** `OSErr NetRegisterName(const unsigned char *name, const unsigned char *type, short version, short socketNumber, const char* hint_addr_string)`
- **Purpose:** Publish local player service for discovery; optionally hint to remote address
- **Inputs:** Player name (Pascal string), service type (Pascal string), version, socket port, optional remote hint address (C string)
- **Outputs/Return:** OSErr (0 on success, SDL error code on resolution failure)
- **Side effects:** Publishes service via SSLP; allocates/frees hint address copy; sets sNameRegistered = true
- **Calls:** NetLookup_BuildTypeString, SSLP_Allow_Service_Discovery, SDLNet_ResolveHost (if hint provided), SSLP_Hint_Service_Discovery, strdup, free
- **Notes:** Host field in SSLP_ServiceInstance ignored; truncates name to fit SSLP_MAX_NAME_LENGTH

### NetUnRegisterName
- **Signature:** `OSErr NetUnRegisterName(void)`
- **Purpose:** Stop publishing local player service
- **Inputs:** None
- **Outputs/Return:** OSErr (always 0)
- **Side effects:** Calls SSLP_Disallow_Service_Discovery, sets sNameRegistered = false
- **Calls:** SSLP_Disallow_Service_Discovery

### NetLookupClose
- **Signature:** `void NetLookupClose(void)`
- **Purpose:** Stop service discovery
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Calls SSLP_Stop_Locating_Service_Instances, sets sLookupInProgress = false
- **Calls:** SSLP_Stop_Locating_Service_Instances

### Stub functions
NetLookupRemove, NetLookupInformation, NetLookupUpdate ΓÇö currently empty (marked "not yet implemented")

## Control Flow Notes
Typical flow: `NetLookupOpen_SSLP()` ΓåÆ SSLP fires callbacks as services appear/disappear ΓåÆ `NetLookupClose()` on shutdown. Separately, `NetRegisterName()` publishes local service, and `NetUnRegisterName()` unpublishes before exit. This is part of the multiplayer game initialization/shutdown, likely called from a higher-level network manager.

## External Dependencies
- **SDL_net:** `SDLNet_ResolveHost` (resolve address string for hinting)
- **SSLP_API.h:** `SSLP_Locate_Service_Instances`, `SSLP_Stop_Locating_Service_Instances`, `SSLP_Allow_Service_Discovery`, `SSLP_Disallow_Service_Discovery`, `SSLP_Hint_Service_Discovery`
- **cseries.h:** OSErr type, noErr constant, string functions (memcpy, sprintf, strlen, strdup, free)
- **sdl_network.h:** NetAddrBlock (IPaddress alias), NetEntityName type
