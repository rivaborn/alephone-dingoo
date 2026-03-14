# Source_Files/Network/network_data_formats.cpp

## File Purpose
Implements bidirectional conversion between packed network data formats (wire format) and unpacked in-memory structures for the Aleph One game engine. Ensures consistent byte ordering and padding across all platforms (little-endian and big-endian systems) during network transmission and reception.

## Core Responsibilities
- Convert network packet headers between wire and native formats
- Serialize/deserialize action packets with endianness handling
- Pack/unpack distribution packets for network transmission
- Convert IP address structures (preserving network byte order)
- Handle network audio header serialization
- Maintain byte-order consistency via ValueToStream/StreamToValue utilities
- Provide overloaded `netcpy()` functions for type-safe format conversions

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| NetPacketHeader_NET | struct | Wire-format packet header (raw byte array) |
| NetPacket_NET | struct | Wire-format action packet |
| NetDistributionPacket_NET | struct | Wire-format distribution packet |
| IPaddress_NET | struct | Wire-format IP address (host + port) |
| network_audio_header_NET | struct | Wire-format audio packet header |

## Global / File-Static State
None.

## Key Functions / Methods

### netcpy (NetPacketHeader overloads)
- **Signature:** `netcpy(NetPacketHeader_NET* dest, const NetPacketHeader* src)` and reverse
- **Purpose:** Pack/unpack packet header (tag, sequence number)
- **Inputs:** Source structure (native or _NET format)
- **Outputs/Return:** Destination filled with converted format
- **Side effects:** Advances internal byte stream pointer via ValueToStream/StreamToValue
- **Calls:** ValueToStream, StreamToValue (from Packing.h)
- **Notes:** Asserts payload size equals SIZEOF_NetPacketHeader (6 bytes)

### netcpy (NetPacket overloads)
- **Signature:** `netcpy(NetPacket_NET* dest, const NetPacket* src)` and reverse
- **Purpose:** Pack/unpack action packet (type, player index, time, flags, counts)
- **Inputs:** Source packet structure
- **Outputs/Return:** Destination in wire or native format
- **Side effects:** Byte swaps individual fields; loops through MAXIMUM_NUMBER_OF_NETWORK_PLAYERS
- **Calls:** ValueToStream, StreamToValue
- **Notes:** Asserts final size matches SIZEOF_NetPacket

### netcpy (uint32 array overload)
- **Signature:** `netcpy(uint32* dest, const uint32* src, size_t length)`
- **Purpose:** Convert uint32 arrays for action flags (conditional little-endian only)
- **Inputs:** Source/dest pointers, byte length
- **Outputs/Return:** Converted array
- **Side effects:** None on big-endian systems (delegates to memcpy)
- **Calls:** ListToStream
- **Notes:** Only compiled if ALEPHONE_LITTLE_ENDIAN defined; asserts length is multiple of sizeof(uint32)

### netcpy (NetDistributionPacket overloads)
- **Signature:** `netcpy(NetDistributionPacket_NET* dest, const NetDistributionPacket* src)` and reverse
- **Purpose:** Pack/unpack distribution packet (type, player, data size)
- **Inputs/Outputs/Return/Side effects:** As NetPacketHeader above
- **Calls:** ValueToStream, StreamToValue

### netcpy (IPaddress overloads)
- **Signature:** `netcpy(IPaddress_NET* dest, const IPaddress* src)` and reverse
- **Purpose:** Copy IP address structures without byte swapping
- **Inputs:** Source IPaddress struct (host as 32-bit int, port as 16-bit int)
- **Outputs/Return:** Destination wire format
- **Side effects:** Direct memcpy of raw fields (no endianness conversion)
- **Calls:** memcpy (2 calls: 4 bytes for host, 2 for port)
- **Notes:** IP addresses must always be network byte order; no conversion applied

### netcpy (network_audio_header overloads)
- **Signature:** `netcpy(network_audio_header_NET* dest, const network_audio_header* src)` and reverse
- **Purpose:** Pack/unpack audio packet header (reserved field, flags)
- **Inputs/Outputs/Return/Side effects:** Similar to NetPacketHeader
- **Calls:** ValueToStream, StreamToValue
- **Notes:** Asserts size equals SIZEOF_network_audio_header (8 bytes)

## Control Flow Notes
Not applicableΓÇöthis is a pure utility/serialization library. Functions are invoked when:
- **Sending:** Native structures ΓåÆ _NET format via sender's netcpy
- **Receiving:** _NET format ΓåÆ native structures via receiver's netcpy
No implicit control flow; all conversions are explicit call sites.

## External Dependencies
- **config.h**: Provides build configuration defines
- **network_data_formats.h**: Struct definitions and extern function declarations
- **Packing.h**: ValueToStream, StreamToValue, ListToStream macros (endianness aware)
- **ALEPHONE_LITTLE_ENDIAN**: Preprocessor define controlling conditional uint32 array conversion
- **DISABLE_NETWORKING**: Guard disabling entire file if networking is disabled
