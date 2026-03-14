# Source_Files/Misc/macintosh_network.h

## File Purpose
Macintosh-specific networking header providing data structures and function prototypes for AppleTalk/ADSP-based network communication. Defines abstractions for DDP (Datagram Delivery Protocol) frames, packet buffers, and ADSP connection endpoints used by the broader network subsystem.

## Core Responsibilities
- Define platform-specific data structures for DDP frame transmission
- Define packet buffer structures for received DDP datagrams
- Define connection endpoint structures for ADSP (AppleTalk Data Stream Protocol)
- Declare function prototypes for name registration/lookup via AppleTalk
- Declare function prototypes for DDP socket and frame operations
- Declare function prototypes for low-level ADSP connection management

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `DDPFrame` | struct | Encapsulates a single DDP frame with payload, MPP parameter block, and WDS elements for transmission |
| `DDPPacketBuffer` | struct | Holds received DDP datagram data with source address, protocol type, and hop count |
| `ConnectionEnd` | struct | Manages an ADSP connection with CCB reference, socket number, send/recv queues, and parameter block |
| `EntityNamePtr` | typedef | Pointer to AppleTalk entity name |
| `lookupUpdateProcPtr` | typedef | Callback for network entity lookup state changes (insert/remove) |
| `lookupFilterProcPtr` | typedef | Callback to filter entities during lookup |
| `PacketHandlerProcPtr` | typedef | Callback for handling received DDP packets |
| `InitializeListenerProcPtr` | typedef | Initializes DDP listener with handler and buffer |

## Global / File-Static State
None.

## Key Functions / Methods

### NetState
- Signature: `short NetState(void)`
- Purpose: Query current network state
- Inputs: None
- Outputs/Return: Network state code (short)
- Side effects: None
- Calls: None visible
- Notes: Likely used to determine if network is active, connecting, gathering, etc.

### NetSetServerIdentifier
- Signature: `void NetSetServerIdentifier(short identifier)`
- Purpose: Set this machine's server/player identifier
- Inputs: `identifier` ΓÇô numeric identifier for this endpoint
- Outputs/Return: None
- Side effects: Modifies internal server identifier state
- Calls: None visible
- Notes: Called during gathering/connection setup

### NetLookupOpen
- Signature: `OSErr NetLookupOpen(unsigned char *name, unsigned char *type, unsigned char *zone, short version, lookupUpdateProcPtr updateProc, lookupFilterProcPtr filterProc)`
- Purpose: Initiate AppleTalk entity lookup with callbacks for updates and filtering
- Inputs: Name, type, zone (AppleTalk NBP params); version; update and filter callbacks
- Outputs/Return: OS error code
- Side effects: Initiates asynchronous lookup; callbacks invoked as results arrive
- Calls: None visible
- Notes: Version parameter is concatenated to lookup type per Aug 1995 note

### NetDDPOpenSocket, NetDDPCloseSocket
- Signature: `OSErr NetDDPOpenSocket(short *socketNumber, PacketHandlerProcPtr packetHandler)` / `OSErr NetDDPCloseSocket(short socketNumber)`
- Purpose: Register/deregister a DDP socket with packet handler callback
- Inputs: Socket number (output for Open); handler callback
- Outputs/Return: OS error; socket number (via pointer for Open)
- Side effects: Registers handler; subsequent DDP packets trigger callback
- Calls: None visible
- Notes: Handler receives `DDPPacketBufferPtr`

### NetDDPNewFrame, NetDDPDisposeFrame
- Signature: `DDPFramePtr NetDDPNewFrame(void)` / `void NetDDPDisposeFrame(DDPFramePtr frame)`
- Purpose: Allocate and free DDP frame structures
- Inputs: Frame pointer (Dispose)
- Outputs/Return: Allocated frame pointer (New)
- Side effects: Memory allocation/deallocation
- Calls: None visible

### NetDDPSendFrame
- Signature: `OSErr NetDDPSendFrame(DDPFramePtr frame, AddrBlock *address, short protocolType, short socket)`
- Purpose: Transmit a DDP frame to a destination address
- Inputs: Frame; destination address; protocol type; socket
- Outputs/Return: OS error code
- Side effects: Sends network packet
- Calls: None visible

## Control Flow Notes
This header is used during network initialization (NetState transitions) and frame exchange. DDP socket listeners are registered early; name lookup and entity discovery occur during gathering phase. Frame send/receive forms the core data path.

## External Dependencies
- **AppleTalk.h** ΓÇô MPP parameter blocks, NBP entity names, address blocks
- **ADSP.h** ΓÇô TPCCB, DSPParamBlock, connection parameter blocks
- **network.h** ΓÇô Generic network interface (included for cross-platform definitions)

**Platform constraint**: Carbon compatibility disabled (line 34 warning); classic AppleTalk only.
