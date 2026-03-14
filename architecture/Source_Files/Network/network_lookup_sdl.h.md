# Source_Files/Network/network_lookup_sdl.h

## File Purpose
Header declaring SDL-based network service registration and lookup functions for Aleph One's multiplayer networking. Wraps the SSLP (Simple Service Location Protocol) API to provide simpler callback-driven service discovery and publishing on non-AppleTalk networks.

## Core Responsibilities
- Declare service registration/unregistration for network game discovery
- Provide SDL-dialog-friendly lookup interface with callback notifications (found/lost/name-changed)
- Support SSLP hint mechanism for cross-subnet service visibility
- Conditionally expose networking APIs only when `DISABLE_NETWORKING` is not set

## Key Types / Data Structures
None defined here. Delegates to `SSLP_ServiceInstance` (from SSLP_API.h) and `SSLP_Service_Instance_Status_Changed_Callback` typedef.

## Global / File-Static State
None.

## Key Functions / Methods

### NetRegisterName
- Signature: `OSErr NetRegisterName(const unsigned char *name, const unsigned char *type, short version, short socketNumber, const char* hint_addr_string)`
- Purpose: Register a local network service for remote discovery via SSLP, with optional hint address for multi-subnet deployment
- Inputs: Service name (Pstring), type (Pstring), version identifier, socket number, optional hint address string
- Outputs/Return: `OSErr` error code
- Side effects: Registers service with SSLP subsystem
- Calls: Not visible in header (implementation in NETWORK_NAMES.C)

### NetUnRegisterName
- Signature: `OSErr NetUnRegisterName(void)`
- Purpose: Unregister the currently registered service
- Inputs: None
- Outputs/Return: `OSErr` error code
- Side effects: Removes service from SSLP discovery
- Calls: Not visible in header

### NetLookupOpen_SSLP
- Signature: `OSErr NetLookupOpen_SSLP(const unsigned char *type, short version, SSLP_Service_Instance_Status_Changed_Callback foundInstance, SSLP_Service_Instance_Status_Changed_Callback lostInstance, SSLP_Service_Instance_Status_Changed_Callback nameChanged)`
- Purpose: Start searching for remote services of a given type with event callbacks
- Inputs: Service type (Pstring), version, three optional callback function pointers for service state changes
- Outputs/Return: `OSErr` error code
- Side effects: Initiates SSLP service discovery; callbacks may be invoked from different threads as services appear/disappear
- Calls: Wraps SSLP discovery (implementation in NETWORK_LOOKUP.C)
- Notes: Replaces older unused `NetLookupUpdate()`. Renamed interface to clarify SSLP/SDL-dialog compatibility.

### NetLookupClose
- Signature: `void NetLookupClose(void)`
- Purpose: Stop service discovery
- Inputs: None
- Outputs/Return: None
- Side effects: Halts ongoing SSLP queries; invalidates service pointers previously passed to callbacks
- Calls: Not visible in header

## Control Flow Notes
This file is part of the network initialization/configuration layer. Service registration typically occurs during game startup; lookup occurs when browsing for available games. Callbacks integrate with SDL dialog widgets (e.g., `w_found_players`).

## External Dependencies
- `SSLP_API.h` ΓÇô provides `SSLP_Service_Instance_Status_Changed_Callback` typedef and `SSLP_ServiceInstance` struct
- `config.h` ΓÇô for `DISABLE_NETWORKING` guard
- Conditional on `HAVE_SDL_NET` (from config.h)
