# Source_Files/Network/SSLP_API.h

## File Purpose
Public API header for SSLP (Simple Service Location Protocol)ΓÇöa custom service discovery mechanism for finding game servers on networks. Designed for the Aleph One project to enable player-to-server discovery without relying on AppleTalk. Intentionally simplified for game server discovery use cases rather than heavyweight enterprise deployment.

## Core Responsibilities
- Define the `SSLP_ServiceInstance` data structure for advertising services
- Provide service location/discovery APIs for clients seeking remote services
- Provide service publishing APIs for servers advertising themselves
- Support callback-based notification of service state changes (found, lost, name changed)
- Abstract over single-threaded and multi-threaded implementations
- Integrate with SDL_net for cross-platform UDP-based communication

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `SSLP_ServiceInstance` | struct | Describes a discoverable service: type name, instance name, and network address (host + port) |
| `SSLP_Service_Instance_Status_Changed_Callback` | typedef (function pointer) | Callback signature for service state notifications (found/lost/name-changed events) |

## Global / File-Static State
None.

## Key Functions / Methods

### SSLP_Pump
- Signature: `void SSLP_Pump(void)`
- Purpose: Provide processing time to SSLP in non-threaded implementations; threaded implementations handle their own timing internally
- Inputs: None
- Outputs/Return: None
- Side effects: May invoke registered callbacks from within this call; updates internal service tracking state
- Calls: (Not visible in header)
- Notes: **Critical in single-threaded mode.** Recommended frequency: ~1ΓÇô5 second intervals. More frequent calls ΓåÆ more responsive discovery.

### SSLP_Locate_Service_Instances
- Signature: `void SSLP_Locate_Service_Instances(const char* inServiceType, SSLP_Service_Instance_Status_Changed_Callback inFoundCallback, SSLP_Service_Instance_Status_Changed_Callback inLostCallback, SSLP_Service_Instance_Status_Changed_Callback inNameChangedCallback)`
- Purpose: Begin searching for available service instances of a given type; report discoveries/losses/name changes via callbacks
- Inputs: Service type string; three optional callback pointers (found, lost, name-changed)
- Outputs/Return: None
- Side effects: Begins broadcasting discovery requests; invokes callbacks (potentially in different thread than caller); maintains internal tracking of discovered services
- Calls: (Implementation detail)
- Notes: **Only one search can be active at a time** in current implementation. Callbacks may fire in different threads if multi-threaded. Service instance pointers remain valid until lost callback fires. Identical instances with different names trigger name-changed callback.

### SSLP_Stop_Locating_Service_Instances
- Signature: `void SSLP_Stop_Locating_Service_Instances(const char* inServiceType)`
- Purpose: Halt an active service location search
- Inputs: Service type (currently ignored; implementation stops all ongoing searches)
- Outputs/Return: None
- Side effects: Stops all discovery broadcasts; invalidates all previously reported service instance pointers; no lost callbacks are fired after this returns
- Calls: (Implementation detail)
- Notes: **ERROR if no search is active.** Parameter is ignored; all searches are stopped regardless.

### SSLP_Allow_Service_Discovery
- Signature: `void SSLP_Allow_Service_Discovery(const SSLP_ServiceInstance* inServiceInstance)`
- Purpose: Publish a local service instance so remote clients can discover it via SSLP queries
- Inputs: Service instance (type, name, port; "host" field ignored)
- Outputs/Return: None
- Side effects: Begins responding to discovery requests; copies service data internally
- Calls: (Implementation detail)
- Notes: **Only one service can be published at a time** in current implementation. Port is on local machine. Instance data is copied; caller may free original.

### SSLP_Hint_Service_Discovery
- Signature: `void SSLP_Hint_Service_Discovery(const SSLP_ServiceInstance* inServiceInstance, const IPaddress* inRemoteHost)`
- Purpose: Announce a service instance to a specific remote host/port (useful for cross-broadcast-domain connectivity)
- Inputs: Service instance; remote target address (must be network byte order)
- Outputs/Return: None
- Side effects: Calls `SSLP_Allow_Service_Discovery()` if not already called; begins periodic unicast announcements to the remote host
- Calls: `SSLP_Allow_Service_Discovery()` (if needed)
- Notes: Target port defaults to SSLP_PORT if 0. Subsequent calls change hint behavior. Data is copied internally.

### SSLP_Disallow_Service_Discovery
- Signature: `void SSLP_Disallow_Service_Discovery(const SSLP_ServiceInstance* inServiceInstance)`
- Purpose: Stop publishing a service instance and halt any associated hints
- Inputs: Service instance pointer (currently ignored; implementation unpublishes all)
- Outputs/Return: None
- Side effects: Stops responding to discovery requests; stops hinting if active; invalidates internal service record
- Calls: (Implementation detail)
- Notes: **ERROR if no service is published.** Parameter is ignored; all published services are stopped.

## Control Flow Notes
This API sits at the network layer during **game initialization and multiplayer lobby phases**. In non-threaded mode, `SSLP_Pump()` must be called regularly (e.g., during the main event loop) to drive discovery and trigger callbacks. In threaded mode, background threads handle timing autonomously. Expected flow:
1. **Init**: Call `SSLP_Pump()` periodically (or rely on threads)
2. **Search phase**: `SSLP_Locate_Service_Instances()` + callbacks populate available servers
3. **Publish phase**: `SSLP_Allow_Service_Discovery()` + optional `SSLP_Hint_Service_Discovery()` makes this instance discoverable
4. **Cleanup**: `SSLP_Stop_Locating_Service_Instances()` and `SSLP_Disallow_Service_Discovery()` to shut down

## External Dependencies
- `SDL_net.h` ΓÇö Cross-platform UDP/IP networking primitives (`IPaddress` type)
- `config.h` ΓÇö Build-time configuration flags; guards entire header with `!DISABLE_NETWORKING`

**Notes**: Callbacks may execute in arbitrary threads if threading is enabled. Service instance pointers are valid only during their advertised lifetime (until lost callback or explicit stop). No guarantee of service existence at advertised addressΓÇöSSLP is best-effort hints only.
