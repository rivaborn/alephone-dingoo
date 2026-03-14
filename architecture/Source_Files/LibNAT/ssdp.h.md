# Source_Files/LibNAT/ssdp.h

## File Purpose
Header file declaring the SSDP (Simple Service Discovery Protocol) interface for LibNAT. Provides a function to send SSDP discover requests to network devices (typically routers) and receive their responses for device discovery and NAT traversal purposes.

## Core Responsibilities
- Declare the SSDP discovery entry point
- Define the interface for sending SSDP requests over the network
- Specify memory management contract for responses (caller frees)

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### LNat_Ssdp_Discover
- Signature: `int LNat_Ssdp_Discover(const char * search_target, char ** response);`
- Purpose: Send an SSDP discover request and retrieve the response from discovered devices
- Inputs:
  - `search_target`: C string specifying the UPnP device/service type to search for (e.g., "ssdp:all")
  - `response`: pointer to char pointer; upon return, points to allocated buffer containing response data
- Outputs/Return:
  - `int`: status code (exact semantics not documented in header; likely 0 for success, non-zero for error)
  - `response`: pointer-to-pointer populated with allocated response buffer
- Side effects:
  - Network I/O: sends UDP multicast SSDP request, receives unicast response
  - Heap allocation: allocates memory for response buffer
  - Likely blocks until response received or timeout
- Calls: Not inferable from this file (definition elsewhere)
- Notes:
  - Caller must free response buffer via `free()`
  - Typical SSDP discovery for UPnP/IGD device enumeration (NAT-related)
  - No include guard details suggest this is a simple public API

## Control Flow Notes
Not inferable from this file; declaration only.

## External Dependencies
- Standard C library (`free()` mentioned in comments)
- SSDP/UPnP network protocol (implementation elsewhere)
