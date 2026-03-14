# tools/MapChunkerMain.cpp

## File Purpose
A standalone Mac utility tool that manipulates resource/chunk conversions in Marathon map files (WAD format). Provides a menu-driven interface for reporting on map contents, moving resources into chunks, and extracting chunks back to resources.

## Core Responsibilities
- Macintosh event loop and menu-bar management (Carbon/Classic Mac APIs)
- Reading and writing WAD (wad_data) files with header/directory structures
- Converting between macOS resource fork format and WAD chunk format for PICT (images), CLUT (color tables), sound, and text resources
- Picture and color-table conversion using QuickTime APIs
- File I/O abstraction through FileSpecifier and OpenedFile wrappers
- Reporting chunk/resource inventory to text files

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `WadContainer` | struct | RAII wrapper for `wad_data*` pointers; manages creation/destruction |
| `InFileData` | struct | Encapsulates input WAD file state: file handle, header, directory indices |
| `OutFileData` | struct | Encapsulates output WAD file state: file handle, header, accumulated directory entries |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `SavedCLUT` | `CTabHandle` | file-static | Cached color table from most recently loaded PICT |
| `GrayCLUT` | `CTabHandle` | file-static | Fallback 8-bit grayscale color table (lazily created) |
| `HasQuicktime` | `bool` | file-static | Capability flag: QuickTime and Movie Toolbox available |
| `HasNavServices` | `bool` | file-static | Capability flag: Navigation Services (file dialogs) available |

## Key Functions / Methods

### main
- **Signature:** `void main(void)`
- **Purpose:** Application entry point; initializes Mac toolbox, sets up menu bar, runs event loop.
- **Inputs:** None
- **Outputs/Return:** None (exits via `ExitToShell()`)
- **Side effects:** Initializes cursor, creates menu bar, enters blocking event loop; calls Macintosh toolbox functions
- **Calls:** `InitCursor()`, `InitMacServices()`, `GetNewMBar()`, `SetMenuBar()`, `AppendResMenu()`, `DrawMenuBar()`, `WaitNextEvent()`, `HandleEvent()`
- **Notes:** Runs `while(true)` with event mask `mDownMask | keyDownMask | autoKeyMask`; exits only via menu command

### HandleEvent, HandleMouseDown, HandleKey
- **Signature:** `void HandleEvent(EventRecord& Event)`, `void HandleMouseDown(EventRecord& Event)`, `void HandleKey(EventRecord& Event)`
- **Purpose:** Dispatch mouse and keyboard events to appropriate menu handlers
- **Inputs:** EventRecord (raw platform event)
- **Outputs/Return:** None
- **Calls:** `FindWindow()`, `MenuSelect()`, `MenuKey()`, `HandleMenuEvent()`

### ListChunks
- **Signature:** `void ListChunks()`
- **Purpose:** Read a WAD file and write a text report of all chunks (tags) and resources to a file.
- **Inputs:** User dialogs for input WAD and output report file
- **Outputs/Return:** Creates text file with inventory
- **Side effects:** Opens WAD and resource fork; changes default Mac directory; writes file
- **Calls:** `InFileData::Open()`, `InFileData::GetWad()`, `OpenedResourceFile::Open()`, `OpenedResourceFile::GetTypeList()`, `OpenedResourceFile::GetIDList()`, Macintosh file/resource APIs
- **Notes:** Uses `fprintf()` to text file; iterates wad tags and resource IDs

### AddChunks
- **Signature:** `void AddChunks()`
- **Purpose:** Convert resource fork entries (PICT, CLUT, sound, text) into WAD chunks and write a new chunked map file.
- **Inputs:** User dialogs for input WAD and output WAD
- **Outputs/Return:** Creates new WAD file with resources moved into chunks
- **Side effects:** Reads input WAD and resource fork; writes output WAD; manages SavedCLUT across iterations
- **Calls:** `InFileData::Open()`, `OutFileData::Open()`, `LoadResourceIntoArray()`, `append_data_to_wad()`, `OutFile.AddWad()`, `OutFile.Finish()`
- **Notes:** Handles resource-to-chunk mapping; uses binary search (`FindInIDList()`) to match resource IDs to wad indices; complex dual-iteration for remaining resources

### ExtractChunks
- **Signature:** `void ExtractChunks()`
- **Purpose:** Extract chunks from a WAD file and move them into the resource fork of a new map file.
- **Inputs:** User dialogs for input and output WAD
- **Outputs/Return:** Creates new WAD file with resource fork containing extracted resources
- **Side effects:** Reads input WAD; creates output WAD; creates and opens resource fork
- **Calls:** `InFileData::Open()`, `OutFileData::Open()`, `OutFileData::Spec.Open(OutRes, true)`, `LoadDataIntoResource()`, `FSpCreateResFile()`, Macintosh resource APIs
- **Notes:** Preliminary pass loads CLUT before PICT so pictures can use the color table

### MakePicture, LoadPictureIntoArray
- **Signature:** `Handle MakePicture(byte *Data, int Length)`, `void LoadPictureIntoArray(Handle DHdl, vector<uint8>& Data)`
- **Purpose:** Convert between raw pixel data (with bounds and color depth header) and QuickTime PICT handles; supports 8/16/24/32-bit formats.
- **Inputs:** Raw pixel buffer with 10-byte header (Rect + uint16 ColorDepth) OR PICT handle
- **Outputs/Return:** PICT Handle OR filled vector with pixel data + header
- **Side effects:** Allocates GWorlds (off-screen graphics contexts); uses QuickTime; SavedCLUT used for 8-bit pictures
- **Calls:** `QTNewGWorldFromPtr()` / `NewGWorld()`, `OpenCPicture()`, `CopyBits()`, `DrawPicture()`, `GetGWorldPixMap()`, `DisposeGWorld()`, `GetPictInfo()`
- **Notes:** Assumes big-endian byte order; row bytes alignment mask `0x7fff` used to ignore topmost bit

### MakeCLUT, LoadCLUTIntoArray
- **Signature:** `Handle MakeCLUT(byte *Data, int Length)`, `void LoadCLUTIntoArray(Handle DHdl, vector<uint8>& Data)`
- **Purpose:** Convert between raw CLUT data (6-byte header + 256 colors ├ù 6 bytes each) and Macintosh ColorTable handles.
- **Inputs:** Raw CLUT buffer OR CTabHandle
- **Outputs/Return:** CTabHandle OR filled vector with CLUT data
- **Side effects:** Allocates/deallocates memory
- **Calls:** `NewHandleClear()`, `DisposeHandle()`
- **Notes:** Byte layout differs between format: input is 6 uint8s per color, output is 8 uint8s (includes index)

## Control Flow Notes
This is a single-threaded event-driven Mac application. Execution follows:
1. **Init phase:** `InitCursor()` ΓåÆ `InitMacServices()` ΓåÆ menu setup
2. **Event loop:** `WaitNextEvent()` with 2-tick timeout ΓåÆ `HandleEvent()` dispatch
3. **Shutdown:** Implicit via `ExitToShell()` from File > Quit menu

File operations (ListChunks, AddChunks, ExtractChunks) are blocking and invoked directly from menu handlers. No background processing; each operation runs synchronously.

## External Dependencies
- **Includes:** `<Carbon.h>`, `<Movies.h>` (QuickTime); `cseries.h`, `crc.h`, `map.h`, `editor.h`, `FileHandler.h`, `wad.h`
- **Defined elsewhere:** `wad_data`, `wad_header`, `tag_data`, `open_wad_file_for_reading()`, `read_wad_header()`, `read_indexed_wad_from_file()`, `append_data_to_wad()`, `write_wad()`, `write_wad_header()`, `calculate_wad_length()`, Macintosh ToolBox functions (`Gestalt()`, `EnterMovies()`, `MenuSelect()`, etc.)
- **Macintosh APIs:** Resource Manager, File Manager (HGetVol, HSetVol, FSpCreateResFile), QuickTime (QTNewGWorldFromPtr, OpenCPicture), Navigation Services (optional)
