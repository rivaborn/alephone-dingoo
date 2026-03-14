# Source_Files/Network/network_capabilities.cpp

## File Purpose
Defines static string constants for network capability flags used in multiplayer protocol negotiation. These constants represent feature identifiers that gatherers and joiners exchange to negotiate compatible protocol versions and optional features (Lua, Speex, zipped data, etc.).

## Core Responsibilities
- Initializes static const string constants that serve as capability identifiers
- Provides a centralized registry of capability flag names for network versioning
- Conditionally compiled when networking is enabled (guarded by `DISABLE_NETWORKING`)
- Part of a capability-negotiation system for game synchronization and optional features

## Key Types / Data Structures
None defined in this file (types defined in header: `Capabilities` class inheriting from `map<string, uint32>`).

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `Capabilities::kGameworld` | `const string` | static (class member) | Capability flag for PRNG/physics protocol compatibility |
| `Capabilities::kStar` | `const string` | static | Star network protocol version flag |
| `Capabilities::kRing` | `const string` | static | Ring network protocol version flag |
| `Capabilities::kLua` | `const string` | static | Lua scripting support flag |
| `Capabilities::kSpeex` | `const string` | static | Speex audio codec support flag |
| `Capabilities::kGatherable` | `const string` | static | Joiner capability flag (can be gathered into session) |
| `Capabilities::kZippedData` | `const string` | static | Compressed data support flag |
| `Capabilities::kNetworkStats` | `const string` | static | Network diagnostics support flag |

## Key Functions / Methods
None. This is a constant definition file.

## Control Flow Notes
This file contains only static initialization with no runtime control flow. The constants are accessed at runtime by other network code (likely during capability negotiation handshakes between game clients) to determine compatible protocol versions.

## External Dependencies
- `config.h` ΓÇö build-time configuration (defines `DISABLE_NETWORKING`)
- `network_capabilities.h` ΓÇö forward declares `Capabilities` class and string type aliases
- `<string>` (C++ stdlib) ΓÇö for `std::string` type

**Notes:**
- File is wrapped in `#if !defined(DISABLE_NETWORKING)` guard; entire file is dead code if networking is disabled.
- Version constants (e.g., `kGameworldVersion = 1`) are defined in the header; this file only defines the corresponding string key names.
- The `Capabilities` class uses `map<string, uint32>` to pair these string keys with version numbers at runtime.
