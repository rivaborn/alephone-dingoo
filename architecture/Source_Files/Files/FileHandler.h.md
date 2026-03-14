# Source_Files/Files/FileHandler.h

## File Purpose
Platform-agnostic abstraction layer for file, directory, and resource management in the Aleph One game engine. Provides unified C++ classes that wrap platform-specific I/O (MacOS resource forks vs. SDL/standard filesystem) to insulate the engine from platform details.

## Core Responsibilities
- Abstract low-level file operations (read, write, seek, close) via `OpenedFile`
- Manage in-memory resources (loading, unloading, access) via `LoadedResource`
- Handle resource fork files on MacOS via `OpenedResourceFile`
- Specify and manipulate file paths with type/creator awareness via `FileSpecifier`
- Provide directory specifications for MacOS volumes/parent IDs via `DirectorySpecifier`
- Support dialog-based file selection for opening/saving
- Enable file discovery, enumeration, and metadata queries (dates, types, free space)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `OpenedFile` | class | Low-level file handle wrapping (read/write/close); platform-specific: MacOS refnum or SDL_RWops |
| `LoadedResource` | class | In-memory resource holder with auto-cleanup; MacOS handles or malloc'd pointers |
| `OpenedResourceFile` | class | Resource fork file manager; supports push/pop for MacOS's global resource chain; resource check/get by type+ID |
| `FileSpecifier` | class | High-level file path with type/creator support; wraps MacOS FSSpec or SDL string paths |
| `DirectorySpecifier` | class | MacOS-only directory specification (vRefNum + parent ID); treated as alias to FileSpecifier on SDL |
| `dir_entry` | struct | SDL directory listing entry (name, size, is_directory, is_volume, modification date) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `RefNum_Closed` | const short | global | Symbolic constant (-1) for closed file refnum on MacOS |
| `isCurrentRefRWOps` | static bool | class static (OpenedResourceFile) | Tracks whether the current resource fork is an SDL_RWops (MacOS/Carbon specific) |

## Key Functions / Methods

### OpenedFile::IsOpen / Close / Read / Write
- Signature: `bool IsOpen(); bool Close(); bool Read(int32 Count, void *Buffer); bool Write(int32 Count, void *Buffer);`
- Purpose: Query open state and perform raw I/O operations
- Inputs: Read/Write take byte count and buffer pointer
- Outputs/Return: bool success flag
- Side effects: Modify file pointer, underlying file contents; auto-closes on destruction
- Calls: Platform-specific MacOS/SDL I/O (not visible in this file)
- Notes: Abstraction hides refnum/SDL_RWops management; fork offset/length tracked for resource forks

### OpenedFile::GetPosition / SetPosition / GetLength / SetLength
- Signature: `bool GetPosition(int32& Position); bool SetPosition(int32 Position); bool GetLength(int32& Length); bool SetLength(int32 Length);`
- Purpose: Query and manipulate file pointer and size
- Inputs: Position or Length by reference or value
- Outputs/Return: bool success; Position/Length filled by reference
- Side effects: Modify file pointer or file size
- Calls: Direct platform file APIs
- Notes: MacOS and SDL variants; SetLength may truncate/expand file

### LoadedResource::IsLoaded / GetPointer / SetData
- Signature: `bool IsLoaded(); void *GetPointer(bool DoDetach = false); void SetData(void *data, size_t length);`
- Purpose: Manage in-memory resource lifetime and access
- Inputs: Pointer and length for SetData; DoDetach flag for GetPointer
- Outputs/Return: IsLoaded bool; void* pointer (optionally detached)
- Side effects: Unload on destruction or explicit Unload(); SetData transfers ownership
- Calls: Detach() to release without deallocating
- Notes: MacOS locks resource handles; DoDetach=true transfers ownership to caller (skip deallocation)

### OpenedResourceFile::Push / Pop / Get / Check
- Signature: `bool Push(); bool Pop(); bool Check(uint32 Type, int16 ID); bool Get(uint32 Type, int16 ID, LoadedResource& Rsrc);`
- Purpose: Manage resource fork as current (MacOS) and query/load resources by type+ID
- Inputs: Type (4-char code, uint32) and ID (int16)
- Outputs/Return: bool success; Get fills Rsrc object
- Side effects: Push/Pop manipulate global resource chain (MacOS); Get loads into Rsrc
- Calls: SetResLoad (MacOS), FOUR_CHARS_TO_INT macro for 4-char to uint32 conversion
- Notes: Overloads accept char parameters for portability; Check does not require Push/Pop; SetResLoad left true

### FileSpecifier::SetNameWithPath / Create / Open
- Signature: `bool SetNameWithPath(const char *NameWithPath); bool Create(Typecode Type); bool Open(OpenedFile& OFile, bool Writable=false);`
- Purpose: Set file path (from root or search path), create new file, open for I/O
- Inputs: Unix-style path (/ separator; : translated on MacOS); Typecode for create; Writable flag for open
- Outputs/Return: bool success; Open fills OFile object
- Side effects: Create allocates disk; Open initializes refnum/SDL_RWops
- Calls: Platform-specific filesystem APIs
- Notes: SetNameWithPath searches data dirs on SDL; Create respects type suffixes or typecode attributes

### FileSpecifier::ReadDialog / WriteDialog / WriteDialogAsync
- Signature: `bool ReadDialog(Typecode Type, const char *Prompt=NULL); bool WriteDialog(Typecode Type, const char *Prompt=NULL, const char *DefaultName=NULL); bool WriteDialogAsync(Typecode Type, char *Prompt=NULL, char *DefaultName=NULL);`
- Purpose: Launch file selection UI for reading or writing
- Inputs: Typecode filter, optional prompt and default name
- Outputs/Return: bool success; FileSpecifier set to chosen path
- Side effects: Modal dialog (WriteDialogAsync is async, allowing sound to continue)
- Calls: Platform UI libraries (not visible)
- Notes: Async variant used for save games to avoid audio stalls

### FileSpecifier::Exists / GetDate / GetType / IsDir
- Signature: `bool Exists(); TimeType GetDate(); Typecode GetType(); bool IsDir();` (IsDir SDL-only)
- Purpose: Query file metadata and existence
- Inputs: None (state from SetNameWithPath or constructor)
- Outputs/Return: bool/Typecode result; GetDate fills TimeType
- Side effects: None (read-only queries)
- Calls: Platform filesystem stat/query APIs
- Notes: GetType returns _typecode_unknown if unrecognized; IsDir SDL-only

### FileSpecifier::CopyContents / Exchange / Delete
- Signature: `bool CopyContents(FileSpecifier& File); bool Exchange(FileSpecifier& File); bool Delete();`
- Purpose: Duplicate file data, atomically swap contents (for safe saves), or remove file
- Inputs: Target FileSpecifier for Copy/Exchange
- Outputs/Return: bool success
- Side effects: Allocate/modify/delete disk data
- Calls: Direct filesystem operations
- Notes: Exchange useful for atomic save patterns (write to temp, exchange with original)

### FileSpecifier::GetName / GetPath / GetFreeSpace
- Signature: `void GetName(char *Name) const; const char *GetPath(void) const; bool GetFreeSpace(uint32& FreeSpace);`
- Purpose: Extract filename, full path, or disk space availability
- Inputs: Name buffer (256 bytes max) or none
- Outputs/Return: Filled buffers or pointers; FreeSpace by reference
- Side effects: None (queries)
- Calls: Platform APIs
- Notes: GetPath returns string pointer (SDL) or requires separate call (MacOS)

### FileSpecifier (SDL-specific): SetToLocalDataDir / SetToPreferencesDir / SetToSavedGamesDir / AddPart / ReadDirectory
- Signature: `void SetToLocalDataDir(); void SetToPreferencesDir(); void SetToSavedGamesDir(); void AddPart(const string &part); bool ReadDirectory(vector<dir_entry> &vec);`
- Purpose: Navigate to standard user directories, build paths incrementally, list directory contents
- Inputs: Part (path component) for AddPart
- Outputs/Return: ReadDirectory fills vector of dir_entry; others modify this object
- Side effects: Canonicalize path on SetTo*; ReadDirectory allocates vector
- Calls: Platform path resolution APIs
- Notes: AddPart and operator+= compose paths; ReadDirectory sorts directories before files

## Control Flow Notes
**Initialization/Game Startup**: FileSpecifier used to locate scenario files, physics files, shapes, sounds, and preferences during engine init.

**Runtime**: LoadedResource and OpenedResourceFile support dynamic asset loading (e.g., textures, sounds). OpenedFile supports save/load game serialization. FileSpecifier dialog methods invoked when user opens/saves manually.

**Platform Abstraction**: MacOS code branches on refnums and FSSpec; SDL code uses string paths and SDL_RWops. Conditional compilation (`#ifdef mac`, `#ifdef SDL`) isolates platform differences. DirectorySpecifier is a true class on MacOS but aliased to FileSpecifier on SDL, minimizing branching.

## External Dependencies
- `tags.h` ΓÇô typecode definitions and helper macros (FOUR_CHARS_TO_INT, Typecode enum)
- `<stddef.h>` ΓÇô size_t
- `<time.h>` ΓÇô time_t (TimeType, presumably)
- `<vector>` ΓÇô STL vector for directory listings and resource lists
- `<string>` ΓÇô STL string (SDL only)
- `<SDL.h>` ΓÇô SDL_RWops file handle abstraction
- `<Carbon/Carbon.h>` ΓÇô MacOS Carbon framework (MacOS only)
- `<windows.h>` ΓÇô Windows API macros (Win32 only; undef GetFreeSpace, CreateDirectory to avoid conflicts)
- Platform-specific I/O not visible in this header (OSErr, FSSpec, refnum semantics defined elsewhere)
