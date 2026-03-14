# Source_Files/Network/Metaserver/metaserver_messages.cpp

## File Purpose
Implements serialization and deserialization (deflation/inflation) of metaserver protocol messages for Aleph One's networking subsystem. Handles encoding/decoding of player information, game descriptions, room listings, chat, and authentication data for TCP communication with metaserver.

## Core Responsibilities
- Serialize message objects to binary streams for network transmission (deflation)
- Deserialize binary data into message objects (inflation)
- Encode/decode player metadata (colors, status, names, teams, ranks)
- Process game descriptions with scenario info, map checksums, and plugin data
- Manage room listings with address/port information
- Support chat and private messaging serialization
- Convert Lua script filenames to human-readable game type strings
- Define static room name lookups and protocol constants

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `sRoomNames` | static char* array | Predefined metaserver room names lookup table |
| `GameDescription` | struct (from header) | Game state metadata (map, difficulty, player count, scenario, plugins) |
| `MetaserverPlayerInfo` | class (from header) | Player record with rank, admin flags, colors, status |
| `RoomDescription` | class | Room metadata (ID, player count, game count, server address, type) |
| `GameListEntry` | nested struct | Game listing with ID, IP, port, time remaining, description |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `sRoomNames` | `const char*[]` | static | 30 hardcoded metaserver room names |
| `kRoomNameCount` | `const int` | static | Array size (30) |
| `kServiceName` | `const char*` | static | Protocol service identifier: "MARATHON" |
| `kKeyLength` | `const int` | static | Encryption key size: 16 bytes |
| `kCRYPT_*` enums | enum | static | Encryption mode constants (plaintext, simple, roomserver) |
| `kSTATE_*` enums | enum | static | Player state flags (awake, away) |
| `kPlatform*` enums | enum | static | Platform identifiers (macintosh, windows, other) |
| `kPlayerStatus` | `static const uint16` | static | Default player state: awake |
| `kPlayerIcon` | `static const uint8` | static | Default icon ID: 0 |
| `kAlephOneClientVersion` | `static const uint16` | static | Client version: 5000 |

## Key Functions / Methods

### write_padded_bytes
- Signature: `static void write_padded_bytes(AOStream& inStream, const char* inBytes, size_t inByteCount, size_t inTargetLength)`
- Purpose: Write bytes to stream with zero-padding to fixed length
- Inputs: stream, byte buffer, byte count, target padded length
- Outputs/Return: void; modifies stream
- Side effects: writes to AOStream
- Calls: `inStream.write()`, `inStream << uint8`

### write_string / write_padded_string
- Signature: `static void write_string(AOStream&, const char*); void write_padded_string(AOStream&, const char*, size_t)`
- Purpose: Write null-terminated string or padded string to stream
- Inputs: stream, string pointer/ref, optional target length
- Outputs/Return: void
- Side effects: writes to stream
- Calls: `strlen()`, `write_padded_bytes()`

### read_string / read_padded_string
- Signature: `static const string read_string(AIStream&); static const string read_padded_string(AIStream&, size_t)`
- Purpose: Read null-terminated or fixed-length string from stream
- Inputs: stream, optional length
- Outputs/Return: std::string
- Side effects: reads from AIStream
- Calls: stream `>>`, `read()`

### get_metaserver_player_color (overloads)
- Signature: `void get_metaserver_player_color(size_t colorIndex, uint16* color); void get_metaserver_player_color(rgb_color, uint16*)`
- Purpose: Convert engine color representation to metaserver RGB triplet (uint16[3])
- Inputs: color index or rgb_color struct, output buffer pointer
- Outputs/Return: void; fills uint16 array [R, G, B]
- Side effects: calls `_get_player_color()`
- Notes: Metaserver uses separate uint16 per channel

### write_player_aux_data
- Signature: `void write_player_aux_data(AOStream& out, string name, const string& team, bool away, const string& away_message)`
- Purpose: Encode player metadata (name, team, colors, status) to stream
- Inputs: stream, player name, team name, away flag, away message
- Outputs/Return: void
- Side effects: writes to stream; reads global preferences; modifies name if away
- Calls: `get_metaserver_player_color()`, `write_padded_bytes()`, `write_string()`
- Notes: Prepends away_message prefix to name if away=true; uses custom colors if `network_preferences->use_custom_metaserver_colors`

### LoginAndPlayerInfoMessage::reallyDeflateTo
- Signature: `void reallyDeflateTo(AOStream&) const`
- Purpose: Serialize login message with platform, service name, date/time, username, and player data
- Inputs: output stream
- Outputs/Return: void
- Side effects: writes to stream
- Calls: `write_padded_string()`, `write_player_aux_data()`, `strlen()`

### SaltMessage::reallyInflateFrom
- Signature: `bool reallyInflateFrom(AIStream&)`
- Purpose: Deserialize encryption salt and type from stream
- Inputs: input stream
- Outputs/Return: bool (always true)
- Side effects: fills `m_encryptionType`, `m_salt[16]`
- Calls: `AIStream::read()`

### RoomDescription::read / operator<<
- Signature: `void RoomDescription::read(AIStream&); ostream& operator<<(ostream&, const RoomDescription&)`
- Purpose: Deserialize room metadata; format for output
- Inputs: stream or ostream
- Outputs/Return: void or ostream
- Side effects: fills room struct; reads from stream
- Calls: `inStream >>`, `inStream.read()`, `inStream.ignore()`

### GameDescription operator>> / operator<<
- Signature: `AIStream& operator>>(AIStream&, GameDescription&); AOStream& operator<<(AOStream&, const GameDescription&)`
- Purpose: Serialize/deserialize game description with plugin-aware field handling
- Inputs: stream, game description
- Outputs/Return: stream reference
- Side effects: reads/writes complex nested structures with conditional plugin data
- Calls: stream `>>`, `<<`, `read()`, `ignore()`; `read_padded_string()`, `write_padded_string()`
- Notes: Plugin flags (0x1, 0x2) control which optional fields are present; status flags encode closed/running state

### GameListMessage::GameListEntry::game_string / format_for_chat
- Signature: `string game_string() const; string format_for_chat(const string& player_name) const`
- Purpose: Convert game type to display name; format game listing for chat output
- Inputs: optional player name
- Outputs/Return: formatted string
- Side effects: none (queries game description)
- Calls: `lua_to_game_string()`, `TS_GetCString()`, `ostringstream`
- Notes: Custom games use Lua script name; standard games look up in string resources

### lua_to_game_string
- Signature: `static string lua_to_game_string(const std::string& lua)`
- Purpose: Strip `.lua`/`.txt` extension and convert underscores to spaces
- Inputs: Lua filename
- Outputs/Return: formatted game name
- Side effects: none
- Calls: `boost::algorithm::ends_with()`, `string::resize()`

### PrivateMessage / ChatMessage constructors & methods
- Signature: `PrivateMessage(uint32 senderID, const string& senderName, uint32 selectedID, const string& message); ChatMessage(...)`
- Purpose: Initialize message with sender/recipient info and fetch player color
- Inputs: sender/recipient IDs, names, message text
- Outputs/Return: void (constructor)
- Side effects: calls `get_metaserver_player_color()`
- Notes: Private messages track directed flag (0x1); chat messages broadcast to all

### MetaserverPlayerInfo constructor
- Signature: `MetaserverPlayerInfo(AIStream& inStream)`
- Purpose: Parse player info record from stream (verb, rank, colors, status, name/team)
- Inputs: input stream
- Outputs/Return: void
- Side effects: fills all member fields via stream reads
- Calls: `inStream >>`, `inStream.read()`, `inStream.ignore()`, `read_string()`
- Notes: Reads admin flags, rank, icon, status bytes; interprets status & 0x1 as away flag

## Control Flow Notes
- **Deflation path** (clientΓåÆserver): Message object ΓåÆ `reallyDeflateTo()` ΓåÆ AOStream ΓåÆ binary bytes
- **Inflation path** (serverΓåÆclient): binary bytes ΓåÆ AIStream ΓåÆ `reallyInflateFrom()` ΓåÆ Message object
- Login sequence: `LoginAndPlayerInfoMessage` ΓåÆ `SaltMessage` ΓåÆ `LoginSuccessfulMessage` with handoff token
- Game listing updates: `GameListMessage` contains vector of `GameListEntry`, each with `GameDescription`
- Player updates: `PlayerListMessage` inflates stream into vector of `MetaserverPlayerInfo` records
- Conditional serialization: `GameDescription` plugin flags gate optional scenario/physics/options fields

## External Dependencies
- **AStream.h**: `AIStream`, `AOStream` (typed binary stream I/O with `>>`, `<<` operators)
- **Message.h**: `Message`, `SmallMessageHelper`, `DatalessMessage<T>` base classes
- **SDL_net.h**: `IPaddress`, `TCPsocket` network types
- **Boost**: `algorithm::ends_with()`, `algorithm::to_lower_copy()` string utilities
- **Game headers**: `preferences.h`, `shell.h` (color/player prefs), `map.h` (TICKS_PER_SECOND), `network_dialogs.h`, `TextStrings.h` (game type strings), `Scenario.h` (scenario metadata)
- **Defined elsewhere**: `_get_player_color()`, `player_preferences`, `network_preferences`, `get_player_color()`, `TS_GetCString()`, `Scenario::instance()`

---
**Notes:**
- File is conditionally compiled (`#if !defined(DISABLE_NETWORKING)`)
- Heavy use of static helper functions for stream I/O; no class-level state mutation beyond message members
- Metaserver protocol uses fixed-size padded fields mixed with variable-length null-terminated strings
- Room/game lookups index into static tables; name resolution is ID-based for robustness
