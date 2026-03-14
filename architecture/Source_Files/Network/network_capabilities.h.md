# Source_Files/Network/network_capabilities.h

## File Purpose
Defines a versioning and capability declaration system for Aleph One's network layer, allowing gatherers (server) and joiners (clients) to advertise and negotiate supported protocols and data formats. Versions track PRNG/physics, network protocols, scripting, audio streaming, and data compression.

## Core Responsibilities
- Define version constants for network components (gameworld, star/ring protocols, Lua, Speex, etc.)
- Provide a `Capabilities` class that acts as a type-safe map of capability name ΓåÆ version number
- Store static string constants identifying each capability feature
- Enforce bounds checking on capability key size (max 1024 chars)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `capabilities_t` | typedef | Type alias for `map<string, uint32>` storing capability nameΓÇôversion pairs |
| `Capabilities` | class | Inherits from `capabilities_t`; wraps map with bounds-checked array access |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `kGameworldVersion` | static const int | class | Version for PRNG and physics engine (v1) |
| `kStarVersion` | static const int | class | Version for star (server) network protocol (v6) |
| `kRingVersion` | static const int | class | Version for ring (peer-to-peer) protocol (v2) |
| `kLuaVersion` | static const int | class | Version for Lua scripting support (v2) |
| `kSpeexVersion` | static const int | class | Version for Speex audio codec (v1) |
| `kGatherableVersion` | static const int | class | Version for joiner capability response (v1) |
| `kZippedDataVersion` | static const int | class | Version for compressed map/lua/physics (v1) |
| `kNetworkStatsVersion` | static const int | class | Version for latency/jitter/error reporting (v1) |
| `kGameworld`, `kStar`, `kRing`, `kLua`, `kSpeex`, `kGatherable`, `kZippedData`, `kNetworkStats` | static const string | class | Capability feature names (keys for the map) |

## Key Functions / Methods

### operator[]
- **Signature:** `uint32& operator[](const string& k)`
- **Purpose:** Type-safe, bounds-checked access to capability versions.
- **Inputs:** `k` ΓÇô capability name (string key)
- **Outputs/Return:** Reference to the version number (uint32).
- **Side effects:** Inserts key into map if not present (std::map behavior); asserts key length < 1024.
- **Calls:** `capabilities_t::operator[]`
- **Notes:** Asserts on oversized keys; allows read/write of capability versions.

## Control Flow Notes
This is a configuration/utility header with no active control flow. It is referenced during network session initialization to check peer version compatibility and negotiate supported features before game state sync.

## External Dependencies
- `<map>`, `<string>` ΓÇô STL containers and strings
- `cseries.h` ΓÇô platform/compiler compatibility layer
- `config.h` ΓÇô build configuration (disable networking flag)
