# Expat/xmlparse/hashtable.cpp

## File Purpose
Hash table implementation using linear probing with automatic resizing. Provides key-value storage for fast lookups (typically used by the Expat XML parser for entity and namespace resolution).

## Core Responsibilities
- Initialize and destroy hash table structures
- Look up entries by key with optional creation of new entries
- Automatically grow table when load factor exceeds 50%
- Iterate over populated entries in the table
- Perform string key comparison and hashing

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| HASH_TABLE | struct | Container: vector of entries, size, usage counters |
| NAMED | struct | Hash table entry; at minimum has `KEY name` field |
| HASH_TABLE_ITER | struct | Iterator state (current pointer and end boundary) |
| KEY | typedef | String key (const char*, wchar_t*, or unsigned short* depending on XML_UNICODE config) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| INIT_SIZE | macro | file | Initial table capacity: 64 buckets |
| keyeq | static function | file | Compare two keys for equality |
| hash | static function | file | Hash function (shift-and-add: h*33 + char) |

## Key Functions / Methods

### lookup
- **Signature:** `NAMED *lookup(HASH_TABLE *table, KEY name, size_t createSize)`
- **Purpose:** Find existing entry or create new one by key; grow table if needed.
- **Inputs:** 
  - `table`: hash table (may be uninitialized if size==0)
  - `name`: key to look up
  - `createSize`: if non-zero, allocate entry of this size; if 0, search-only mode
- **Outputs/Return:** Pointer to NAMED entry if found/created; NULL on allocation failure or not found
- **Side effects:** 
  - Initializes table (calloc INIT_SIZE) if size==0
  - Allocates new entry via calloc(1, createSize) when creating
  - Grows table 2├ù and rehashes all entries when used ΓëÑ usedLim
  - Increments table->used counter on creation
- **Calls:** hash(), keyeq(), calloc(), free()
- **Notes:** Linear probing with backward search (wraps at size boundary). Requires power-of-2 size for `& (size - 1)` bitwise modulo.

### hashTableInit
- **Signature:** `void hashTableInit(HASH_TABLE *p)`
- **Purpose:** Zero-initialize a hash table structure.
- **Inputs:** `p` - pointer to uninitialized HASH_TABLE
- **Outputs/Return:** void
- **Side effects:** Sets size, used, usedLim, v to 0
- **Notes:** Required before first use of lookup().

### hashTableDestroy
- **Signature:** `void hashTableDestroy(HASH_TABLE *table)`
- **Purpose:** Free all allocated memory within a hash table.
- **Inputs:** `table` - initialized HASH_TABLE
- **Outputs/Return:** void
- **Side effects:** Frees each non-NULL entry, then frees vector itself
- **Notes:** Does not reset HASH_TABLE structure; caller must manage it.

### hashTableIterInit
- **Signature:** `void hashTableIterInit(HASH_TABLE_ITER *iter, const HASH_TABLE *table)`
- **Purpose:** Initialize iterator for table traversal.
- **Inputs:** `iter` - uninitialized iterator; `table` - source table
- **Outputs/Return:** void
- **Side effects:** Sets iter->p to table->v; iter->end to table->v + table->size

### hashTableIterNext
- **Signature:** `NAMED *hashTableIterNext(HASH_TABLE_ITER *iter)`
- **Purpose:** Retrieve next populated entry in iteration.
- **Inputs:** `iter` - iterator state
- **Outputs/Return:** Pointer to next NAMED; NULL when iterator reaches end
- **Side effects:** Advances iter->p; skips empty buckets
- **Notes:** Safe to call repeatedly until NULL returned.

## Control Flow Notes
Not part of an event loop; used as a utility library by the Expat XML parser. Typical usage: hashTableInit() during parser creation ΓåÆ lookup() called for each entity/namespace reference during parsing ΓåÆ hashTableDestroy() during parser cleanup. Table grows dynamically as needed.

## External Dependencies
- **Headers:** `xmldef.h` (for conditional memory management: stdlib.h, Windows API, or NSPR), `hashtable.h` (type definitions)
- **Memory:** malloc/calloc/free (may be redirected to HeapAlloc or NSPR depending on platform macros)
- **String:** Relies on null-terminated strings (KEY type)
