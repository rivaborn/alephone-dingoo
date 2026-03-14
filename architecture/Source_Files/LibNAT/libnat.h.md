# Source_Files/LibNAT/libnat.h

## File Purpose
Public API header for LibNAT, a UPnP library for discovering and controlling Internet Gateway Devices. Provides functions for NAT traversal, port mapping, and retrieving public IP addresses to enable network connectivity behind NAT firewalls (e.g., for peer-to-peer file transfers).

## Core Responsibilities
- Discover UPnP-enabled IGDs on the local network
- Query public IP address from discovered IGDs
- Establish and remove port mappings on IGDs for TCP/UDP protocols
- Manage lifecycle of controller objects
- Report library errors to the caller

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `UpnpController` | struct | Opaque handle to an IGD; stores control URL and service type for UPnP communication |

## Global / File-Static State
None.

## Key Functions / Methods

### LNat_Upnp_Discover
- **Signature:** `int LNat_Upnp_Discover(UpnpController ** c)`
- **Purpose:** Search the local network for UPnP-enabled IGDs that support WANIPConnection or WANPPPConnection services.
- **Inputs:** Pointer to uninitialized `UpnpController` pointer.
- **Outputs/Return:** Populates `*c` with allocated `UpnpController`; returns 0 on success, negative error code otherwise.
- **Side effects:** Allocates memory for `UpnpController`; performs network discovery (SSDP multicast).
- **Calls:** Not inferable from this file.
- **Notes:** Caller must eventually call `LNat_Upnp_Controller_Free` to release allocated memory.

### LNat_Upnp_Get_Public_Ip
- **Signature:** `int LNat_Upnp_Get_Public_Ip(const UpnpController * controller, char * public_ip, int public_ip_size)`
- **Purpose:** Query the discovered IGD for its public IP address.
- **Inputs:** Valid `UpnpController` from `LNat_Upnp_Discover`; pre-allocated buffer `public_ip` of size `public_ip_size`.
- **Outputs/Return:** Writes null-terminated IP string to `public_ip` (truncated to `public_ip_size - 1` if needed); returns 0 on success.
- **Side effects:** Network I/O (SOAP request to IGD).
- **Calls:** Not inferable from this file.
- **Notes:** Caller responsible for buffer allocation; output is always null-terminated.

### LNat_Upnp_Set_Port_Mapping
- **Signature:** `int LNat_Upnp_Set_Port_Mapping(const UpnpController * controller, const char * ip_map, short int port_map, const char * protocol)`
- **Purpose:** Create a port mapping on the IGD to forward inbound traffic.
- **Inputs:** Valid controller; target IP on local network (NULL to auto-detect); port number; protocol ("TCP" or "UDP").
- **Outputs/Return:** Returns 0 on success, negative error code on failure.
- **Side effects:** Network I/O (SOAP request to IGD); modifies IGD routing state.
- **Calls:** Not inferable from this file.
- **Notes:** If `ip_map` is NULL, implementation attempts automatic IP detection.

### LNat_Upnp_Remove_Port_Mapping
- **Signature:** `int LNat_Upnp_Remove_Port_Mapping(const UpnpController * controller, short int port_map, const char * protocol)`
- **Purpose:** Delete a previously established port mapping on the IGD.
- **Inputs:** Valid controller; port number; protocol ("TCP" or "UDP").
- **Outputs/Return:** Returns 0 on success, negative error code otherwise.
- **Side effects:** Network I/O (SOAP request to IGD); modifies IGD routing state.
- **Calls:** Not inferable from this file.
- **Notes:** No IP parameter (unlike `Set_Port_Mapping`); parameter naming inconsistency with `Set` variant (`portMap` vs. `port_map`).

### LNat_Upnp_Controller_Free
- **Signature:** `int LNat_Upnp_Controller_Free(UpnpController ** controller)`
- **Purpose:** Deallocate a `UpnpController` object created by `LNat_Upnp_Discover`.
- **Inputs:** Pointer to `UpnpController` pointer.
- **Outputs/Return:** Returns 0 on success.
- **Side effects:** Frees heap memory; nullifies or invalidates the controller.
- **Calls:** Not inferable from this file.
- **Notes:** Must be called exactly once per discovered controller to avoid memory leaks.

### LNat_Print_Error
- **Signature:** `void LNat_Print_Error(int error)`
- **Purpose:** Print human-readable description of an error code to stderr.
- **Inputs:** Error code (negative integer from `error.h` defines or 0 for OK).
- **Outputs/Return:** None (void); side effect is stderr output.
- **Side effects:** I/O to stderr.
- **Calls:** Likely wraps `LNat_Print_Internal_Error` (defined in `error.h`).
- **Notes:** Useful for debugging; complements error codes returned by other functions.

## Control Flow Notes
Typical usage is initialization ΓåÆ query ΓåÆ mapping ΓåÆ cleanup:
1. Call `Discover()` once at startup to locate an IGD.
2. Optionally call `Get_Public_Ip()` to retrieve the external address.
3. Call `Set_Port_Mapping()` one or more times to enable inbound connections.
4. Call `Remove_Port_Mapping()` as needed to free ports.
5. Call `Controller_Free()` at shutdown.

No frame/render loop affinity; this is a passive utility library for network configuration.

## External Dependencies
- **Includes:** `"error.h"` ΓÇö defines all error code constants and `LNat_Print_Internal_Error()`.
- **Conditionally compiled:** `LIBNAT_API` macro for DLL export (currently disabled via `#if 0`).
- **C++ compatibility:** Wrapped in `extern "C"` block.
- **UPnP protocol:** Implementation (not in this file) must perform SSDP discovery, XML parsing, and SOAP communication; likely depends on socket, HTTP, and XML parsing libraries.
