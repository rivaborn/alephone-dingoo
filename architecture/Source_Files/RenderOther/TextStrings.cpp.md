# Source_Files/RenderOther/TextStrings.cpp

## File Purpose
Implements a text string collection system that replaces MacOS STR# resources with a portable, XML-loadable string repository. Strings are organized by ID (resource-like) and index, stored as MacOS Pascal strings (length byte + chars), and can be retrieved as either Pascal or C strings.

## Core Responsibilities
- Manage multiple string collections (sets) indexed by ID using a linked list
- Provide Pascal and C string storage/retrieval operations
- Dynamically grow string arrays on demand (doubling strategy)
- Load strings from XML documents via callback-based parser
- Handle cleanup and deletion at granular (string, set) and bulk (all sets) levels

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `StringSet` | class (private) | Container for strings under one ID; manages dynamic array and linked list node |
| `XML_StringSetParser` | class (private) | Callback parser for `<stringset>` elements; locates/creates StringSet by ID |
| `XML_StringParser` | class (private) | Callback parser for `<string>` elements; converts UTF-8 content to Pascal strings |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `StringSetRoot` | `StringSet*` | static | Root pointer of linked list of all string sets |
| `StringSetParser` | `XML_StringSetParser` | static | Reusable parser instance for stringset elements |
| `StringParser` | `XML_StringParser` | static | Reusable parser instance for string elements |
| `IndexNotFound` | `const char[]` | static | Error message constant |

## Key Functions / Methods

### TS_PutString
- Signature: `void TS_PutString(short ID, short Index, unsigned char *String)`
- Purpose: Insert or replace a Pascal string in the repository
- Inputs: ID (set identifier), Index (position in set), String (pointer to Pascal string: length byte + chars)
- Outputs/Return: None
- Side effects: Calls `FindStringSet()` (creates set if absent), allocates/deallocates heap memory via `StringSet::Add()`
- Calls: `FindStringSet()`, `StringSet::Add()`
- Notes: String is copied; caller retains ownership of input

### TS_PutCString
- Signature: `void TS_PutCString(short ID, short Index, const char *String)`
- Purpose: Insert or replace a C string (null-terminated) by converting to Pascal format
- Inputs: ID, Index, C-string pointer
- Outputs/Return: None
- Side effects: Allocates intermediate buffer (256 bytes stack), calls `StringSet::Add()` with Pascal format
- Calls: `FindStringSet()`, `strlen()`, `memcpy()`, `StringSet::Add()`
- Notes: Truncates to 255 chars if input exceeds length byte capacity

### TS_GetString
- Signature: `unsigned char *TS_GetString(short ID, size_t Index)`
- Purpose: Retrieve a Pascal string from the repository
- Inputs: ID, Index
- Outputs/Return: Pointer to Pascal string (length byte + chars + null), or NULL if not found
- Side effects: None (read-only)
- Calls: Linear search through StringSetRoot linked list, then `StringSet::GetString()`
- Notes: Caller must not delete returned pointer; caller should treat as read-only

### TS_GetCString
- Signature: `char *TS_GetCString(short ID, size_t Index)`
- Purpose: Retrieve a Pascal string as a C string (pointer to char data after length byte)
- Inputs: ID, Index
- Outputs/Return: Pointer into Pascal string at first character (skipping length byte), or NULL if not found
- Side effects: None
- Calls: `TS_GetString()`
- Notes: Relies on null terminator added by `StringSet::Add()`; pointer arithmetic assumes valid Pascal string layout

### TS_IsPresent
- Signature: `bool TS_IsPresent(short ID)`
- Purpose: Check whether a string set with given ID exists
- Inputs: ID
- Outputs/Return: true if found, false otherwise
- Side effects: None
- Calls: Linear linked-list search
- Notes: Does not create a new set (unlike `FindStringSet()`)

### TS_CountStrings
- Signature: `size_t TS_CountStrings(short ID)`
- Purpose: Count contiguous stored strings in a set starting from index 0
- Inputs: ID
- Outputs/Return: Count of valid (non-null) strings, or 0 if set not found
- Side effects: None
- Calls: Linear linked-list search, then `StringSet::CountStrings()`
- Notes: Stops at first gap (null entry)

### TS_DeleteString, TS_DeleteStringSet, TS_DeleteAllStrings
- **TS_DeleteString**: Remove single string at Index; calls `StringSet::Delete()`
- **TS_DeleteStringSet**: Unlink and deallocate entire set from linked list
- **TS_DeleteAllStrings**: Walk linked list and delete all sets; reset `StringSetRoot` to NULL
- Side effects: Deallocate heap memory
- Calls: `StringSet` destructor, `delete` operator

### TS_GetParser
- Signature: `XML_ElementParser *TS_GetParser()`
- Purpose: Return the global XML parser for loading strings from `<stringset>` elements
- Inputs: None
- Outputs/Return: Pointer to static `StringSetParser` instance with `StringParser` registered as child
- Side effects: Adds `StringParser` as child of `StringSetParser` (idempotent registration intended)
- Calls: `XML_ElementParser::AddChild()`
- Notes: Caller must not delete returned pointer

### StringSet::Add (private)
- Signature: `void StringSet::Add(size_t Index, unsigned char *String)`
- Purpose: Store a Pascal string at given index, growing array if necessary
- Inputs: Index, String (Pascal format)
- Outputs/Return: None
- Side effects: Allocates/deallocates heap; may reallocate entire string array with doubling strategy
- Calls: `objlist_clear()`, `objlist_copy()`, `memcpy()`, `delete[]`, `new[]`
- Notes: Owns copy of input string; replaces existing string at Index if present. **Bug**: `Index < 0` check is always false (size_t is unsigned)

### XML_StringParser callbacks
- **Start()**: Reset index-present and string-loaded flags
- **HandleAttribute()**: Parse `index="N"` attribute; store in `Index` member
- **HandleString()**: Convert UTF-8 content to Pascal string via `DeUTF8_Pas()`, call `StringSetParser.CurrStringSet->Add()`
- **AttributesDone()**: Validate that index was provided
- **End()**: If no string content was provided, add empty string (length byte = 0)

## Control Flow Notes
- **Initialization**: Not explicit; string sets created on-demand by `FindStringSet()` when `TS_Put*()` is called
- **XML Loading**: Parser callbacks are invoked by external XML parser infrastructure (not shown). `TS_GetParser()` returns root parser; child callbacks populate string data
- **Shutdown**: Not automatically triggered; caller must invoke `TS_DeleteAllStrings()` for cleanup
- **Retrieval**: Direct linked-list O(n) lookup by ID; no caching

## External Dependencies
- **Includes**: `<string.h>`, `cseries.h` (SDL, MacOS type shims), `TextStrings.h` (own header), `XML_ElementParser.h` (XML framework)
- **Helper functions** (defined elsewhere): `objlist_clear()`, `objlist_copy()`, `StringsEqual()`, `ReadInt16Value()`, `DeUTF8_Pas()` (XML_ElementParser.h)
- **MacOS types**: `Str255` (256-byte Pascal string buffer), `int16`, `uint16`, `size_t`

---

**Notes**: 
- Code is O(n) in number of string sets; acceptable for small counts but could optimize with hash table
- Repeated linked-list traversal patterns are not DRY
- Signed/unsigned mismatch in `Index < 0` conditions (index is `size_t`)
- No thread safety; assumes single-threaded access
