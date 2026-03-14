# Source_Files/Network/network_messages.cpp

## File Purpose
Implements serialization and deserialization ("deflate" and "inflate") operations for network game messages used in multiplayer setup and communication. Handles encoding/decoding of player info, game topology, chat, stats, and supports zlib compression for large data transfers.

## Core Responsibilities
- Serialize/deserialize player network data (addresses, identifiers, player info)
- Implement deflate/inflate for 10+ message types (HelloMessage, TopologyMessage, CapabilitiesMessage, etc.)
- Support zlib compression for large payloads (maps, physics, Lua scripts)
- Manage string encoding (C-strings and Pascal strings) in network format
- Convert between in-memory structures and network byte-order wire format

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| NetPlayer | struct | Stores a single networked player's addresses, stream ID, and player data |
| UninflatedMessage | class | Compressed network message payload (used by BigChunkOfZippedDataMessage) |
| Capabilities | (via include) | Key-value pairs for client feature negotiation |
| ClientChatInfo | struct | Chat-relevant metadata (name, color, team) |
| NetworkStats | struct | Per-player latency/jitter/error metrics |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `write_string` | function | static | Null-terminated C-string serialization to AOStream |
| `write_pstring` | function | static | Pascal-string conversion and serialization |
| `read_string` | function | static | Deserialize null-terminated string from AIStream |
| `read_pstring` | function | static | Deserialize Pascal string from AIStream |
| `deflateNetPlayer` | function | static | Serialize NetPlayer addresses and player_data |
| `inflateNetPlayer` | function | static | Deserialize NetPlayer from stream |

## Key Functions / Methods

### deflateNetPlayer (static)
- **Signature:** `void deflateNetPlayer(AOStream& outputStream, const NetPlayer &player)`
- **Purpose:** Serialize a complete NetPlayer structure (addresses, IDs, player metadata) to network byte order.
- **Inputs:** Output stream, NetPlayer reference
- **Outputs/Return:** None (modifies stream)
- **Side effects:** Writes 30+ bytes to output stream; no allocation
- **Calls:** `write_pstring()` (for player name)
- **Notes:** Comment indicates this is a legacy design; any protocol-incompatible changes should use per-message version instead of modifying this shared function.

### inflateNetPlayer (static)
- **Signature:** `void inflateNetPlayer(AIStream& inputStream, NetPlayer &player)`
- **Purpose:** Deserialize network bytes back into NetPlayer structure.
- **Inputs:** Input stream, destination NetPlayer
- **Outputs/Return:** None (populates player struct)
- **Side effects:** Reads ~30 bytes from stream; throws on read failure
- **Calls:** `read_pstring()`
- **Notes:** Mirror of `deflateNetPlayer`; must maintain exact field order.

### BigChunkOfZippedDataMessage::inflateFrom
- **Signature:** `bool inflateFrom(const UninflatedMessage& inUninflated)`
- **Purpose:** Decompress a zlib-encoded message into the message buffer.
- **Inputs:** Compressed UninflatedMessage; first 4 bytes are uncompressed size (uint32 big-endian)
- **Outputs/Return:** `true` on success; `false` on decompression error
- **Side effects:** Allocates temporary vector; calls `uncompress()` from zlib; calls `copyBufferFrom()` to store result
- **Calls:** `uncompress()` (zlib), stream extraction operator
- **Notes:** Returns `false` and logs warning on Z_OK failure; handles zero-length payloads.

### BigChunkOfZippedDataMessage::deflate
- **Signature:** `UninflatedMessage* deflate() const`
- **Purpose:** Compress the message buffer using zlib and wrap in a new UninflatedMessage.
- **Inputs:** None (uses implicit `buffer()` and `length()`)
- **Outputs/Return:** Newly allocated `UninflatedMessage*` with compressed data; `null` on compression failure
- **Side effects:** Allocates UninflatedMessage on heap; allocates temporary compression buffer; calls `compress()`
- **Calls:** `compress()` (zlib), `buffer()`, `length()`, `type()`
- **Notes:** Size estimate uses `length() * 105 / 100 + 12` per zlib spec; handles empty messages.

### AcceptJoinMessage::reallyDeflateTo / reallyInflateFrom
- **Signature:** `void reallyDeflateTo(AOStream&) const` / `bool reallyInflateFrom(AIStream&)`
- **Purpose:** Serialize/deserialize acceptance decision and player info for join responses.
- **Inputs/Outputs:** Accepted flag (Uint8), NetPlayer data
- **Calls:** `deflateNetPlayer()` / `inflateNetPlayer()`

### CapabilitiesMessage::reallyDeflateTo / reallyInflateFrom
- **Signature:** `void reallyDeflateTo(AOStream&) const` / `bool reallyInflateFrom(AIStream&)`
- **Purpose:** Encode/decode client capability map (string key ΓåÆ Uint8 value).
- **Inputs/Outputs:** Capabilities map
- **Calls:** `write_string()` / `read_string()`
- **Notes:** Inflate reads until stream exhaustion; deflate writes all key-value pairs.

### TopologyMessage::reallyDeflateTo / reallyInflateFrom
- **Signature:** `void reallyDeflateTo(AOStream&) const` / `bool reallyInflateFrom(AIStream&)`
- **Purpose:** Serialize complete game topology (tag, player count, all NetPlayer entries, all game settings).
- **Inputs/Outputs:** NetTopology (tag, player_count, nextIdentifier, game_data, MAXIMUM_NUMBER_OF_NETWORK_PLAYERS NetPlayer entries)
- **Side effects:** Writes ~500+ bytes per message
- **Calls:** `deflateNetPlayer()` / `inflateNetPlayer()` ├ù MAXIMUM_NUMBER_OF_NETWORK_PLAYERS
- **Notes:** Largest message type; serializes game_info (seed, type, limits, difficulty, level name, etc.).

### Other Message reallyDeflateTo / reallyInflateFrom implementations
Brief helper summaries:
- **HelloMessage:** Version string only
- **JoinerInfoMessage:** stream_id, player name (pstring), version, color, team
- **ChangeColorsMessage:** Two int16 fields (color, team)
- **ClientInfoMessage:** stream_id, action, color, team, name string
- **NetworkChatMessage:** senderID, target, targetID, chat text (fixed 1024-byte buffer)
- **NetworkStatsMessage:** Vector of latency/jitter/error triplets, read until stream exhaustion
- **ServerWarningMessage:** uint16 reason code, string warning message

## Control Flow Notes
- **Initialization phase:** HelloMessage and JoinerInfoMessage exchanged first
- **Topology/setup phase:** CapabilitiesMessage and TopologyMessage define the game session
- **In-game phase:** NetworkChatMessage, NetworkStatsMessage, and ChangeColorsMessage communicated as-needed
- **Data distribution phase:** Map/Physics/Lua messages (via BigChunkOfZippedDataMessage subclasses) sent once per session
- **Deflate is called before transmission**, inflate is called after reception. Error handling on decompression logs warning but continues.

## External Dependencies
- **AStream.h** ΓÇö `AIStream`, `AIStreamBE`, `AOStream`, `AOStreamBE` for endian-aware serialization
- **zlib.h** ΓÇö `compress()`, `uncompress()` for compression/decompression
- **network_messages.h** ΓÇö Message class declarations (SmallMessageHelper base, message types)
- **network_private.h** ΓÇö `NetPlayer`, `NetTopology`, `game_info`, `player_info`, `ClientChatInfo`, `NetworkStats` structures
- **network_data_formats.h** ΓÇö Data format constants and netcpy functions
- **Logging.h** ΓÇö `logWarning1()` for error reporting
- **cseries.h** ΓÇö Platform-specific types and macros (pstring conversion functions `a1_p2cstr`, `a1_c2pstr`)
