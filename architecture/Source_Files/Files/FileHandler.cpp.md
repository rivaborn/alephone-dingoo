# Source_Files/Files/FileHandler.cpp

## File Purpose
MacOS implementation of file and resource handling for the Aleph One game engine. Provides abstractions for low-level file I/O, resource fork management, directory navigation, and file dialogs, encapsulating MacOS FSSpec/resource fork APIs.

## Core Responsibilities
- File and resource file lifecycle management (open, close, read, write, position)
- MacOS resource fork operations (Push/Pop resource stack, load/check resources)
- Path parsing with Unix-style separators (/) translated to MacOS conventions
- Directory traversal and specification management
- File metadata queries (type, date, modification time, free space)
- File operations: creation, deletion, copying (with safe save via exchange), existence checks
- Navigation Services-based file dialogs (read, write, async write)
- MacBinary format detection and fork offset handling

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `OpenedFile` | class | Wraps file reference number and error state; reads/writes data at positioned offsets |
| `OpenedResourceFile` | class | Manages MacOS resource fork stack (global resource state); loads/checks typed resources |
| `LoadedResource` | class | RAII wrapper for Handle-based resource data; auto-unlocks and releases on destruction |
| `DirectorySpecifier` | class | Encapsulates vRefNum + parID for directory identity; navigates subdirectories |
| `FileSpecifier` | class | Encapsulates FSSpec; creates, opens, reads type/date, manages file dialogs |
| `dir_entry` | struct | Portable directory entry (SDL); name, size, is_directory, date fields |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `RootDirectorySet` | bool | static | Flag indicating whether custom root directory has been configured |
| `RootDirectory` | DirectorySpecifier | static | Cached root directory for relative path resolution |
| `OpenedResourceFile::isCurrentRefRWOps` | bool | static | Tracks whether current resource file uses SDL RWops or MacOS ResFile API |

## Key Functions / Methods

### `is_macbinary(short RefNum, int32 &data_length, int32 &rsrc_length)`
- **Signature:** `bool is_macbinary(short RefNum, int32 &data_length, int32 &rsrc_length)`
- **Purpose:** Detect and validate MacBinary format header; extract data and resource fork sizes.
- **Inputs:** File reference number; output parameters for fork sizes.
- **Outputs/Return:** true if valid MacBinary header with correct CRC; populates data_length and rsrc_length.
- **Side effects:** Seeks to file start, reads 128-byte header.
- **Calls:** `SetFPos`, `FSRead`.
- **Notes:** Recognizes up to MacBinary III (0x81); CRC polynomial 0x1021.

### `OpenedFile::Open` / `Close` / `Read` / `Write`
- **Signature:** (various; see FileHandler.h for declarations)
- **Purpose:** Read/write data from file at current position; manage file lifecycle.
- **Inputs:** Count (bytes), Buffer (void*) for Read/Write; Position for SetPosition.
- **Outputs/Return:** bool indicating success; Position output param for GetPosition.
- **Side effects:** Updates RefNum, Err, adjusts position by fork_offset if MacBinary.
- **Calls:** `FSpOpenDF`, `FSClose`, `FSRead`, `FSWrite`, `GetFPos`, `SetFPos`.
- **Notes:** Handles MacBinary fork offsets transparently; errors stored in Err field.

### `OpenedResourceFile::Push` / `Pop`
- **Signature:** `bool Push(); bool Pop();`
- **Purpose:** Save/restore the global MacOS resource file stack state.
- **Inputs:** None (uses implicit RefNum / RWops state).
- **Outputs/Return:** true on success; Err updated.
- **Side effects:** Calls `UseResFile` or `use_res_file`; updates global `isCurrentRefRWOps`.
- **Calls:** `CurResFile`, `cur_res_file`, `UseResFile`, `use_res_file`, `ResError`.
- **Notes:** Dual-path: handles both MacOS classic ResFile and SDL RWops APIs; SavedRefNum/saved_f track prior state.

### `OpenedResourceFile::Get` / `Check`
- **Signature:** `bool Get(uint32 Type, int16 ID, LoadedResource& Rsrc); bool Check(uint32 Type, int16 ID);`
- **Purpose:** Load or check for presence of typed resource in resource fork.
- **Inputs:** Type (OSType), ID (resource ID); Get also takes output LoadedResource ref.
- **Outputs/Return:** true if resource found/loaded; Rsrc.RsrcHandle populated on Get.
- **Side effects:** Calls Push/Pop to manage resource stack; SetResLoad toggles.
- **Calls:** `Push`, `Pop`, `get_1_resource`, `Get1Resource`, `SetResLoad`, `ReleaseResource`.
- **Notes:** Check path disables resource loading to avoid side effects; Rsrc is unloaded first.

### `DirectorySpecifier::SetToSubdirectory`
- **Signature:** `bool SetToSubdirectory(const char *NameWithPath);`
- **Purpose:** Navigate to a subdirectory given a Unix-style path string.
- **Inputs:** Path string with / separators (e.g., "Maps/Official").
- **Outputs/Return:** true on success; updates vRefNum, parID.
- **Side effects:** Sets up FSSpec, calls ParsePath_MacOS; parID updated to target directory.
- **Calls:** `Files_GetRootDirectory`, `ParsePath_MacOS`.
- **Notes:** Starts from root directory; empty path reverts to app parent.

### `FileSpecifier::SetNameWithPath`
- **Signature:** `bool SetNameWithPath(const char *NameWithPath);`
- **Purpose:** Parse a path with directories and filename; set FSSpec accordingly.
- **Inputs:** Full path (e.g., "Scenarios/Marathon.sceA").
- **Outputs/Return:** true on success; updates Spec.
- **Side effects:** Calls ParsePath_MacOS with WantDirectory=false to locate file.
- **Calls:** `Files_GetRootDirectory`, `ParsePath_MacOS`.
- **Notes:** Last path component is treated as filename, not directory.

### `FileSpecifier::Open` (OpenedFile variant)
- **Signature:** `bool FileSpecifier::Open(OpenedFile& OFile, bool Writable=false);`
- **Purpose:** Open a file for reading or writing; detect and handle MacBinary wrapping.
- **Inputs:** OpenedFile reference; Writable flag.
- **Outputs/Return:** true on success; OFile.RefNum populated.
- **Side effects:** Calls FSpOpenDF; on read, checks for MacBinary and sets fork_offset/fork_length; calls SetPosition(0).
- **Calls:** `ResolveFile`, `FSpOpenDF`, `is_macbinary`, `SetPosition`.
- **Notes:** Writable opens skip MacBinary detection; errors stored in OFile.Err and posted via set_game_error.

### `FileSpecifier::GetType`
- **Signature:** `Typecode FileSpecifier::GetType();`
- **Purpose:** Determine file type via creator code, extension, or MacBinary magic numbers.
- **Inputs:** None (uses Spec).
- **Outputs/Return:** Typecode (_typecode_scenario, _typecode_unknown, etc.).
- **Side effects:** Opens file to read version/tag if extension unknown; calls FSpGetFInfo.
- **Calls:** `FSpGetFInfo`, `get_typecode_for_file_type`, `Open` (OpenedFile), `Read`.
- **Notes:** Tries type code, then extension, then magic numbers; returns _typecode_unknown on mismatch.

### `FileSpecifier::ReadDialog` / `WriteDialog`
- **Signature:** `bool ReadDialog(Typecode Type, const char *Prompt=NULL); bool WriteDialog(...);`
- **Purpose:** Present macOS Navigation Services file open/save dialogs.
- **Inputs:** Typecode (for filtering file types); optional prompt and default name.
- **Outputs/Return:** true if user confirmed; Spec populated from dialog result.
- **Side effects:** Calls NavGetFile / NavPutFile; flushes event queue; sets up NavTypeListHandle.
- **Calls:** `FlushEvents`, `machine_has_nav_services`, `NavGetFile`, `NavPutFile`, `ExtractSingleItem`.
- **Notes:** Filters by typecode if in range; else allows all types; asserts on !Nav Services (fallback disabled).

### `ParsePath_MacOS`
- **Signature:** `static bool ParsePath_MacOS(const char *NameWithPath, FSSpec &Spec, bool WantDirectory);`
- **Purpose:** Recursively walk directory tree, parsing Unix-style path into vRefNum + parID.
- **Inputs:** Path string; FSSpec with starting directory; WantDirectory flag.
- **Outputs/Return:** true on success; Spec.parID updated (and Spec.name if file found).
- **Side effects:** Iterates filesystem; case-insensitive name comparison via PBGetCatInfo.
- **Calls:** `PBGetCatInfo`, `memcpy`, `memset`.
- **Notes:** Splits on /; translates : to / for MacOS compat; checks file/directory attribute (0x10 bit).

### `FileSpecifier::CopyContents`
- **Signature:** `bool FileSpecifier::CopyContents(FileSpecifier& File);`
- **Purpose:** Safe file copy via temp file + exchange; preserves original on error.
- **Inputs:** Source FileSpecifier (File).
- **Outputs/Return:** true on success; this becomes copy of File.
- **Side effects:** Creates savetemp.dat; copies data fork in 3KB chunks; exchanges and deletes temp.
- **Calls:** `FSpGetFInfo`, `FSpCreate`, `FSpOpenDF`, `SetFPos`, `GetFPos`, `FSRead`, `FSWrite`, `FSClose`, `Exchange`, `Delete`.
- **Notes:** Data fork only (no resource fork); allocates 3KB buffer on heap.

## Control Flow Notes
File I/O lifecycle: `FileSpecifier` ΓåÆ `SetNameWithPath` ΓåÆ `Open(OpenedFile)` ΓåÆ `Read/Write` ΓåÆ `Close`. Resource loading: `FileSpecifier` ΓåÆ `Open(OpenedResourceFile)` ΓåÆ `Push` ΓåÆ `Get/Check` ΓåÆ `Pop` ΓåÆ `Close`. File type detection occurs at Open time for unknown extensions. Root directory is lazily initialized and cached in static state. Navigation Services dialogs block until user responds.

## External Dependencies
- **MacOS APIs:** Carbon/Carbon.h (if EXPLICIT_CARBON_HEADER), FSSpec, FSRef, resource fork (Get1Resource, UseResFile, etc.), Navigation Services (NavGetFile, NavPutFile), Aliases, Folders.
- **Custom types:** `Typecode`, `OSType` (from tags.h); `DirectorySpecifier`, `OpenedFile`, `LoadedResource` (defined in FileHandler.h).
- **Utility functions:** `set_game_error`, `global_idle_proc`, `machine_has_nav_services`, `get_typecode`, `get_all_file_types_for_typecode` (defined elsewhere); `obj_clear`, `obj_copy`, `FOUR_CHARS_TO_INT`, `MIN`, `NONE` (from cseries.h).
- **SDL RWops:** Conditional support for SDL-based resource file I/O as alternative to native MacOS ResFile.
