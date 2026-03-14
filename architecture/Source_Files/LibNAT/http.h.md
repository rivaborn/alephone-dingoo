# Source_Files/LibNAT/http.h

## File Purpose
Header file declaring HTTP client functions for sending GET and POST requests to a router. Provides a high-level interface for constructing HTTP messages, adding headers, executing requests, and managing memory for request/response structures.

## Core Responsibilities
- Generate and destroy HTTP GET request structures
- Generate and destroy HTTP POST request structures with bodies
- Add Request Header Fields and Entity Header Fields to POST messages
- Execute HTTP GET and POST requests to remote servers
- Manage memory allocation/deallocation for request and response buffers

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| GetMessage | struct | Encapsulates HTTP GET request metadata (host, resource, port). Forward-declared; implementation elsewhere. |
| PostMessage | struct | Encapsulates HTTP POST request metadata and headers (host, resource, port, body). Forward-declared; implementation elsewhere. |

## Global / File-Static State
None.

## Key Functions / Methods

### LNat_Generate_Http_Get
- Signature: `int LNat_Generate_Http_Get(const char * host, const char * resource, short int port, GetMessage ** gm)`
- Purpose: Construct a GET request structure from connection parameters.
- Inputs: host (server address), resource (path/URI), port (server port)
- Outputs/Return: Pointer to allocated `GetMessage` via output parameter `gm`; returns OK/error code
- Side effects: Allocates heap memory for `GetMessage` structure
- Calls: (implementation not visible)
- Notes: Caller must invoke `LNat_Destroy_Http_Get` to free memory. Returns error if allocation fails.

### LNat_Destroy_Http_Get
- Signature: `int LNat_Destroy_Http_Get(GetMessage ** gm)`
- Purpose: Free memory allocated by `LNat_Generate_Http_Get`.
- Inputs: Pointer to `GetMessage` pointer
- Outputs/Return: Status code (OK/error)
- Side effects: Deallocates heap memory, nullifies pointer

### LNat_Generate_Http_Post
- Signature: `int LNat_Generate_Http_Post(const char * host, const char * resource, short int port, const char * body, PostMessage ** pm)`
- Purpose: Construct a POST request structure with message body.
- Inputs: host, resource, port, body (message content)
- Outputs/Return: Pointer to allocated `PostMessage` via output parameter `pm`; returns status code
- Side effects: Allocates heap memory for `PostMessage` structure
- Calls: (implementation not visible)
- Notes: Caller may optionally add headers via `LNat_Http_Post_Add_Request_Header` and `LNat_Http_Post_Add_Entity_Header` before sending.

### LNat_Http_Post_Add_Request_Header
- Signature: `int LNat_Http_Post_Add_Request_Header(PostMessage * pm, const char * token, const char * value)`
- Purpose: Append a Request Header Field (HTTP spec header) to a POST message.
- Inputs: PostMessage pointer, header name (token), header value
- Outputs/Return: Status code

### LNat_Http_Post_Add_Entity_Header
- Signature: `int LNat_Http_Post_Add_Entity_Header(PostMessage * pm, const char * token, const char * value)`
- Purpose: Append an Entity Header Field (payload-describing header) to a POST message.
- Inputs: PostMessage pointer, header name, header value
- Outputs/Return: Status code

### LNat_Destroy_Http_Post
- Signature: `int LNat_Destroy_Http_Post(PostMessage ** pm)`
- Purpose: Free memory allocated by `LNat_Generate_Http_Post`.
- Side effects: Deallocates heap, nullifies pointer

### LNat_Http_Request_Get
- Signature: `int LNat_Http_Request_Get(GetMessage * gm, char ** response)`
- Purpose: Execute the HTTP GET request and retrieve server response.
- Inputs: Prepared `GetMessage` structure
- Outputs/Return: Pointer to response buffer via output parameter `response`; returns OK/error code
- Side effects: Network I/O; allocates heap memory for response (caller must `free()`)

### LNat_Http_Request_Post
- Signature: `int LNat_Http_Request_Post(PostMessage * pm, char ** response)`
- Purpose: Execute the HTTP POST request and retrieve server response.
- Inputs: Prepared `PostMessage` structure with optional headers
- Outputs/Return: Pointer to response buffer via output parameter `response`; returns status code
- Side effects: Network I/O; allocates heap memory for response (caller must `free()`)

## Control Flow Notes
Typical usage pattern: **Generate** (GET/POST constructor) ΓåÆ **Optionally add headers** (POST only) ΓåÆ **Execute request** ΓåÆ **Destroy** message structure. Responses are caller-allocated on heap and must be manually freed. Not inferable whether this integrates into a frame loop or init phase from header alone.

## External Dependencies
- Standard C library (likely `stdio.h`, memory functions)
- Network/socket library (implementation file not provided)
- Likely uses UPnP/NAT library context ("LibNAT" suggests NAT traversal utilities)
