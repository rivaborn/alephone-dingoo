# Source_Files/Network/network_lookup.cpp

## File Purpose
Implements asynchronous network entity discovery on classic Macintosh AppleTalk using NBP (Name Binding Protocol). Maintains a persistent, alphabetically-sorted list of discovered game servers/clients with automatic stale-entry timeout, optional per-entity filtering, and caller-supplied change notifications.

## Core Responsibilities
- Initiate and manage asynchronous NBP entity lookups (NetLookupOpen/Close)
- Maintain sorted in-memory cache of discovered entities with persistence tracking
- Extract and insert new entities from NBP results; detect duplicates by network address
- Age out stale entities (not heard from in ~4 seconds); notify caller on removal
- Provide filtering capability via caller-supplied callback (e.g., exclude in-game players)
- Notify caller when entities are added/removed via updateProc callback
- Query and enumerate available AppleTalk zones; identify local zone
- Populate macOS menu UI with zone list

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| NetLookupEntity | struct | Stores discovered entity name, network address (node/socket/net), and persistence counter |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| lookupMPPPBPtr | MPPPBPtr | static | Active NBP lookup parameter block; NULL when idle |
| lookupEntity | NetEntityNamePtr | static | Entity name/type pattern being searched for |
| lookupBuffer | Ptr | static | ~4KB buffer filled by PLookupName with result list |
| lookupCount | short | static | Current number of cached entities |
| lookupEntities | NetLookupEntityPtr | static | Array of discovered entities (up to 40) |
| lookupUpdateProc | lookupUpdateProcPtr | static | Callback (may be NULL) invoked on entity add/remove |
| lookupFilterProc | lookupFilterProcPtr | static | Callback (may be NULL) to accept/reject discovered entities |

## Key Functions / Methods

### NetLookupOpen
- Signature: `OSErr NetLookupOpen(unsigned char *name, unsigned char *type, unsigned char *zone, short version, lookupUpdateProcPtr updateProc, lookupFilterProcPtr filterProc)`
- Purpose: Initiate asynchronous periodic entity discovery
- Inputs: Pascal-string entity name/type to search for; zone ("*" for all); version suffix concatenated to type; optional update and filter callbacks
- Outputs/Return: OSErr (success/memory error)
- Side effects: Allocates lookupMPPPBPtr (~300B), lookupEntity, lookupBuffer (4KB), lookupEntities array (40├ùentry); populates NBP param block; calls PLookupName() asynchronously; stores callbacks in static globals
- Calls: NewPtrClear, psprintf, NBPSetEntity, PLookupName, MemError
- Notes: Asserts no lookup is already active; does not block; type is padded with version number (e.g., "MyType2" for type="MyType", version=2)

### NetLookupClose
- Signature: `void NetLookupClose(void)`
- Purpose: Terminate pending lookup and free all allocated resources
- Inputs: None
- Outputs/Return: None
- Side effects: Creates temp param block, calls PKillNBP to cancel async operation, disposes lookupMPPPBPtr, lookupEntity, lookupBuffer, lookupEntities; nullifies lookupMPPPBPtr
- Calls: PKillNBP, DisposePtr
- Notes: Silently ignores reqAborted/noErr/cbNotFound; other errors logged via dprintf

### NetLookupRemove
- Signature: `void NetLookupRemove(short index)`
- Purpose: Remove entity from cache (manual eviction before stale timeout)
- Inputs: Index into lookupEntities
- Outputs/Return: None
- Side effects: Shifts array elements via BlockMove; decrements lookupCount; invokes updateProc(removeEntity, index) if registered
- Calls: BlockMove, optional lookupUpdateProc
- Notes: Used when entity becomes unresponsive or joins game; caller must pass valid index

### NetLookupInformation
- Signature: `void NetLookupInformation(short index, NetAddrBlock *address, NetEntityName *entity)`
- Purpose: Retrieve cached entity details by index
- Inputs: Index; optional pointers to receive address and entity structures
- Outputs/Return: Populates address/entity parameters (in-place copy)
- Side effects: None
- Calls: None (direct struct copy)
- Notes: Caller may pass NULL for either output; no validation of pointers

### NetLookupUpdate
- Signature: `void NetLookupUpdate(void)`
- Purpose: Poll for completed NBP lookup; process results and manage persistence timeouts
- Inputs: None
- Outputs/Return: None
- Side effects: If PLookupName completed: extracts entities via NBPExtract, applies filterProc, searches sorted list for duplicates (by aNode/aSocket/aNet), inserts new entries at correct alphabetical position, updates persistence of existing entries, removes stale entities (persistence < 0), notifies caller via updateProc, restarts PLookupName
- Calls: NBPExtract, optional lookupFilterProc/lookupUpdateProc, NetLookupRemove, PlookupName, IUCompString
- Notes: Non-blocking; returns immediately if lookup incomplete (ioResult==asyncUncompleted); maintains alphabetical order for efficient binary-search-like insertion; decrements persistence each call; ENTITY_PERSISTENCE=4 (Γëê4 sec at ~1 Hz update rate); detects duplicates before insertion to avoid duplicates; calls removeEntity callback before removal to allow caller cleanup

### NetGetZonePopupMenu
- Signature: `OSErr NetGetZonePopupMenu(MenuHandle menu, short *local_zone)`
- Purpose: Populate Macintosh menu with zone list and check local zone
- Inputs: Menu handle; pointer to receive local zone index (0-based)
- Outputs/Return: OSErr; local_zone set to 1-based menu index on success
- Side effects: Clears all menu items; allocates zone array; sets cursor to watch; calls NetGetZoneList; appends menu items; checks local zone item; restores cursor; disposes temp buffer
- Calls: CountMenuItems, DeleteMenuItem, NewPtr, SetCursor, NetGetZoneList, AppendMenu, SetMenuItemText, CheckItem, DisposePtr, MemError
- Notes: Converts local_zone from 0-based to 1-based for macOS menu indexing; shows watch cursor during network query

### NetGetZoneList
- Signature: `OSErr NetGetZoneList(Str32 *zone_names, short maximum_zone_names, short *zone_count, short *local_zone)`
- Purpose: Enumerate all available AppleTalk zones and identify local zone
- Inputs: Zone name array to fill; max count; pointers to receive actual count and local zone index
- Outputs/Return: OSErr; zone_names populated (up to maximum_zone_names); zone_count set to actual; local_zone set to 0-based index in list
- Side effects: Allocates XPPParamBlk and 578B ZIP buffer; loops calling PBControl(xCall/zipGetZoneList) until zipLastFlag or max reached; calls qsort to alphabetize zones; calls NetGetLocalZoneName to identify local zone; searches for local zone in sorted list
- Calls: NewPtrClear, PBControl, qsort, zone_name_compare, NetGetLocalZoneName, IUCompString, DisposePtr
- Notes: Paces zone retrieval via xppTimeout=4, xppRetry=8; parses PString zone names from buffer; warns (via vwarn) if local zone not found in list

### NetGetLocalZoneName
- Signature: `OSErr NetGetLocalZoneName(Str32 local_zone_name)`
- Purpose: Query the local machine's AppleTalk zone
- Inputs: Pointer to 32-byte string buffer
- Outputs/Return: OSErr; local_zone_name populated with zone name
- Side effects: Allocates XPPParamBlk; executes PBControl(xCall/zipGetMyZone); disposes param block
- Calls: NewPtrClear, PBControl, DisposePtr
- Notes: Uses XPP (experimental protocol) interface; result is a PString (length-prefixed)

## Control Flow Notes

**Initialization:** NetLookupOpen() allocates resources and starts first async PLookupName() call.

**Per-frame loop:** NetLookupUpdate() polls for completion. On completion: extracts new entities from NBP result buffer, filters via caller callback, maintains sorted list (insert/update), ages out stale entries, notifies caller for each add/remove, and re-issues PLookupName(). Non-blocking; no operation if lookup not yet complete.

**Shutdown:** NetLookupClose() cancels pending lookup and deallocates all structures.

**Zone enumeration (independent of entity lookup):** NetGetZoneList() / NetGetZonePopupMenu() query zone list via ZIP protocol, optionally populate menu UI, and identify local zone for zone-aware filtering.

## External Dependencies
- **AppleTalk.h**: NBP (PLookupName, PKillNBP, NBPExtract, NBPSetEntity), ZIP (PBControl), XPP (param blocks)
- **macintosh_network.h**: lookupUpdateProcPtr, lookupFilterProcPtr, NetEntityName, AddrBlock, EntityName types
- **macintosh_cseries.h**: Memory (NewPtrClear, DisposePtr, MemError), assertion, string utilities (psprintf, pstrcpy, IUCompString)
- **stdlib.h**: qsort
- **string.h**: Implicit via pstrcpy
- Macintosh Toolbox: SetCursor, GetCursor, CountMenuItems, DeleteMenuItem, AppendMenu, SetMenuItemText, CheckItem, BlockMove
- Conditional: TEST_MODEM flag substitutes ModemLookup* stubs; DISABLE_NETWORKING disables entire file
